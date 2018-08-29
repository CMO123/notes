# f2fs_fill_super流程
```c
static int f2fs_fill_super(struct super_block *sb, void *data, int silent)
{
	struct f2fs_sb_info *sbi;
	struct f2fs_super_block *raw_super;
	struct inode *root;

try_onemore:
part-1:
	-/为f2fs-specific super block info分配内存
	sbi = kzalloc(sizeof(struct f2fs_sb_info), GFP_KERNEL);
	sbi->sb = sb;
  /* Load the checksum driver */
	sbi->s_chksum_driver = crypto_alloc_shash("crc32", 0, 0);
	-/ 设置block size
	if (unlikely(!sb_set_blocksize(sb, F2FS_BLKSIZE))) {...}
	/从设备中读取super_block信息
	err = read_raw_super_block(sbi, &raw_super, &valid_super_block,
								&recovery);
part-2:
	//	s_fs_info与文件系统类型相关，不同文件系统该信息不同，但都保存到超级块的sb->s_fs_info中.
	// 这部分信息很重要，文件系统中经常用到。
	sb->s_fs_info = sbi;
	sbi->raw_super = raw_super;

	sbi->s_resuid = make_kuid(&init_user_ns, F2FS_DEF_RESUID);
	sbi->s_resgid = make_kgid(&init_user_ns, F2FS_DEF_RESGID);

	/* precompute checksum seed for metadata */
	if (f2fs_sb_has_inode_chksum(sb)){//没有进入条件循环
		sbi->s_chksum_seed = f2fs_chksum(sbi, ~0, raw_super->uuid,
						sizeof(raw_super->uuid));
	}

	/*
	 * The BLKZONED feature indicates that the drive was formatted with
	 * zone alignment optimization. This is optional for host-aware
	 * devices, but mandatory for host-managed zoned block devices.
	 */
#ifndef CONFIG_BLK_DEV_ZONED
	if (f2fs_sb_mounted_blkzoned(sb)) {
		f2fs_msg(sb, KERN_ERR,
			 "Zoned block device support is not enabled\n");
		err = -EOPNOTSUPP;
		goto free_sb_buf;
	}
#endif
	default_options(sbi);
	/* parse mount options */
	options = kstrdup((const char *)data, GFP_KERNEL);
	if (data && !options) {
		err = -ENOMEM;
		goto free_sb_buf;
	}

	err = parse_options(sb, options);
	if (err)
		goto free_options;

	sbi->max_file_blocks = max_file_blocks();
	sb->s_maxbytes = sbi->max_file_blocks <<
				le32_to_cpu(raw_super->log_blocksize);
	sb->s_max_links = F2FS_LINK_MAX;
	get_random_bytes(&sbi->s_next_generation, sizeof(u32));

#ifdef CONFIG_QUOTA
	/*
	Disk quota磁盘配额技术是一种限制文件系统空间使用的技术。在Linux系统中，系统管理员可
	以通过该技术限制其他用户在指定的容量范围内使用文件系统，从而防止个别用户过量使用而
	影响到其他的用户，因此早先的磁盘配额技术都是基于user id和group id实现的。本文介绍的
	project quota技术是社区近来新实现的一种磁盘配额技术，它不再基于用户和组来划分空间，
	而基于project id实现，限额的粒度可以细到某个目录甚至单个文件，实现对文件系统空间布局进行控制。
	*/
	sb->dq_op = &f2fs_quota_operations;
	if (f2fs_sb_has_quota_ino(sb))// 貌似没有进入if
		sb->s_qcop = &dquot_quotactl_sysfile_ops;
	else
		sb->s_qcop = &f2fs_quotactl_ops;
	sb->s_quota_types = QTYPE_MASK_USR | QTYPE_MASK_GRP | QTYPE_MASK_PRJ;

	if (f2fs_sb_has_quota_ino(sbi->sb)) {
		for (i = 0; i < MAXQUOTAS; i++) {
			if (f2fs_qf_ino(sbi->sb, i))
				sbi->nquota_files++;
		}
	}
#endif
	//初始化super_block的s_op项,sb->s_op= &jffs2_super_operations;
	sb->s_op = &f2fs_sops;
#ifdef CONFIG_F2FS_FS_ENCRYPTION
	sb->s_cop = &f2fs_cryptops;
#endif
	sb->s_xattr = f2fs_xattr_handlers;
	sb->s_export_op = &f2fs_export_ops;
	sb->s_magic = F2FS_SUPER_MAGIC;
	sb->s_time_gran = 1;//c/m/atime的粒度
	sb->s_flags = (sb->s_flags & ~SB_POSIXACL) |
		(test_opt(sbi, POSIX_ACL) ? SB_POSIXACL : 0);
	memcpy(&sb->s_uuid, raw_super->uuid, sizeof(raw_super->uuid));
	sb->s_iflags |= SB_I_CGROUPWB;

	/* init f2fs-specific super block info */
	sbi->valid_super_block = valid_super_block;
	mutex_init(&sbi->gc_mutex);
	mutex_init(&sbi->cp_mutex);
	init_rwsem(&sbi->node_write);
	init_rwsem(&sbi->node_change);

	/* disallow all the data/node/meta page writes */
	set_sbi_flag(sbi, SBI_POR_DOING);
	spin_lock_init(&sbi->stat_lock);

	/* init iostat info */
	spin_lock_init(&sbi->iostat_lock);
	sbi->iostat_enable = false;

	for (i = 0; i < NR_PAGE_TYPE; i++) {
		int n = (i == META) ? 1: NR_TEMP_TYPE;
		int j;

		sbi->write_io[i] = f2fs_kmalloc(sbi,
					n * sizeof(struct f2fs_bio_info),
					GFP_KERNEL);
		if (!sbi->write_io[i]) {
			err = -ENOMEM;
			goto free_options;
		}

		for (j = HOT; j < n; j++) {
			init_rwsem(&sbi->write_io[i][j].io_rwsem);
			sbi->write_io[i][j].sbi = sbi;
			sbi->write_io[i][j].bio = NULL;
			spin_lock_init(&sbi->write_io[i][j].io_lock);
			INIT_LIST_HEAD(&sbi->write_io[i][j].io_list);
		}
	}

	init_rwsem(&sbi->cp_rwsem);
	init_waitqueue_head(&sbi->cp_wait);
	init_sb_info(sbi);

	err = init_percpu_info(sbi);
	if (err)
		goto free_bio_info;

	if (F2FS_IO_SIZE(sbi) > 1) {
		sbi->write_io_dummy =
			mempool_create_page_pool(2 * (F2FS_IO_SIZE(sbi) - 1), 0);
		if (!sbi->write_io_dummy) {
			err = -ENOMEM;
			goto free_percpu;
		}
	}

	/* get an inode for meta space */

	sbi->meta_inode = f2fs_iget(sb, F2FS_META_INO(sbi));
	if (IS_ERR(sbi->meta_inode)) {
		f2fs_msg(sb, KERN_ERR, "Failed to read F2FS meta data inode");
		err = PTR_ERR(sbi->meta_inode);
		goto free_io_dummy;
	}

	err = get_valid_checkpoint(sbi);
	if (err) {
		f2fs_msg(sb, KERN_ERR, "Failed to get valid F2FS checkpoint");
		goto free_meta_inode;
	}

	/* Initialize device list */
	err = f2fs_scan_devices(sbi);
	if (err) {
		f2fs_msg(sb, KERN_ERR, "Failed to find devices");
		goto free_devices;
	}

	sbi->total_valid_node_count =
				le32_to_cpu(sbi->ckpt->valid_node_count);
	percpu_counter_set(&sbi->total_valid_inode_count,
				le32_to_cpu(sbi->ckpt->valid_inode_count));
	sbi->user_block_count = le64_to_cpu(sbi->ckpt->user_block_count);
	sbi->total_valid_block_count =
				le64_to_cpu(sbi->ckpt->valid_block_count);
	sbi->last_valid_block_count = sbi->total_valid_block_count;
	sbi->reserved_blocks = 0;
	sbi->current_reserved_blocks = 0;
	limit_reserve_root(sbi);

	for (i = 0; i < NR_INODE_TYPE; i++) {
		INIT_LIST_HEAD(&sbi->inode_list[i]);
		spin_lock_init(&sbi->inode_lock[i]);
	}

	init_extent_cache_info(sbi);

	init_ino_entry_info(sbi);

	/* setup f2fs internal modules */
	err = build_segment_manager(sbi);
	if (err) {
		f2fs_msg(sb, KERN_ERR,
			"Failed to initialize F2FS segment manager");
		goto free_sm;
	}
	err = build_node_manager(sbi);
	if (err) {
		f2fs_msg(sb, KERN_ERR,
			"Failed to initialize F2FS node manager");
		goto free_nm;
	}

	/* For write statistics */
	if (sb->s_bdev->bd_part)
		sbi->sectors_written_start =
			(u64)part_stat_read(sb->s_bdev->bd_part, sectors[1]);

	/* Read accumulated write IO statistics if exists */
	seg_i = CURSEG_I(sbi, CURSEG_HOT_NODE);
	if (__exist_node_summaries(sbi))
		sbi->kbytes_written =
			le64_to_cpu(seg_i->journal->info.kbytes_written);

	build_gc_manager(sbi);

	/* get an inode for node space */
	sbi->node_inode = f2fs_iget(sb, F2FS_NODE_INO(sbi));
	if (IS_ERR(sbi->node_inode)) {
		f2fs_msg(sb, KERN_ERR, "Failed to read node inode");
		err = PTR_ERR(sbi->node_inode);
		goto free_nm;
	}

	err = f2fs_build_stats(sbi);
	if (err)
		goto free_node_inode;

	/* read root inode and dentry */
	/*
	该函数能够查询特定超级块下指定inode number的inode。如果inode number不存在，则创建新的inode，并将inode number赋值给新的inode.
	一般来说系统都用类似的jffs2_iget->iget_locked/ubifs_iget->iget_locked来创建新inode，或查询已存在的inode。
	每个超级块都存在一个root inode,其中结点号是inode->i_ino=1，返回值是超级块的根节点inode。
	*/
	root = f2fs_iget(sb, F2FS_ROOT_INO(sbi));
	if (IS_ERR(root)) {
		f2fs_msg(sb, KERN_ERR, "Failed to read root inode");
		err = PTR_ERR(root);
		goto free_stats;
	}
	if (!S_ISDIR(root->i_mode) || !root->i_blocks || !root->i_size) {
		iput(root);
		err = -EINVAL;
		goto free_node_inode;
	}
	//每个超级块都存在一个root inode对应的目录项dentry,其中结点号是dentry->d_iname="/"
	sb->s_root = d_make_root(root); /* allocate root dentry */
	if (!sb->s_root) {
		err = -ENOMEM;
		goto free_root_inode;
	}

	err = f2fs_register_sysfs(sbi);
	if (err)
		goto free_root_inode;

#ifdef CONFIG_QUOTA
	/*
	 * Turn on quotas which were not enabled for read-only mounts if
	 * filesystem has quota feature, so that they are updated correctly.
	 */
	if (f2fs_sb_has_quota_ino(sb) && !sb_rdonly(sb)) {
		err = f2fs_enable_quotas(sb);
		if (err) {
			f2fs_msg(sb, KERN_ERR,
				"Cannot turn on quotas: error %d", err);
			goto free_sysfs;
		}
	}
#endif
	/* if there are nt orphan nodes free them */
	err = recover_orphan_inodes(sbi);
	if (err)
		goto free_meta;

	/* recover fsynced data */
	if (!test_opt(sbi, DISABLE_ROLL_FORWARD)) {
		/*
		 * mount should be failed, when device has readonly mode, and
		 * previous checkpoint was not done by clean system shutdown.
		 */
		if (bdev_read_only(sb->s_bdev) &&
				!is_set_ckpt_flags(sbi, CP_UMOUNT_FLAG)) {
			err = -EROFS;
			goto free_meta;
		}

		if (need_fsck)
			set_sbi_flag(sbi, SBI_NEED_FSCK);

		if (!retry)
			goto skip_recovery;

		err = recover_fsync_data(sbi, false);
		if (err < 0) {
			need_fsck = true;
			f2fs_msg(sb, KERN_ERR,
				"Cannot recover all fsync data errno=%d", err);
			goto free_meta;
		}
	} else {
		err = recover_fsync_data(sbi, true);

		if (!f2fs_readonly(sb) && err > 0) {
			err = -EINVAL;
			f2fs_msg(sb, KERN_ERR,
				"Need to recover fsync data");
			goto free_meta;
		}
	}
skip_recovery:
	/* recover_fsync_data() cleared this already */
	clear_sbi_flag(sbi, SBI_POR_DOING);

	/*
	 * If filesystem is not mounted as read-only then
	 * do start the gc_thread.
	 */
	if (test_opt(sbi, BG_GC) && !f2fs_readonly(sb)) {
		/* After POR, we can run background GC thread.*/
		err = start_gc_thread(sbi);
		if (err)
			goto free_meta;
	}
	kfree(options);

	/* recover broken superblock */
	if (recovery) {
		err = f2fs_commit_super(sbi, true);
		f2fs_msg(sb, KERN_INFO,
			"Try to recover %dth superblock, ret: %d",
			sbi->valid_super_block ? 1 : 2, err);
	}

	f2fs_join_shrinker(sbi);

	f2fs_msg(sbi->sb, KERN_NOTICE, "Mounted with checkpoint version = %llx",
				cur_cp_version(F2FS_CKPT(sbi)));
	f2fs_update_time(sbi, CP_TIME);
	f2fs_update_time(sbi, REQ_TIME);
	return 0;

free_meta:
#ifdef CONFIG_QUOTA
	if (f2fs_sb_has_quota_ino(sb) && !sb_rdonly(sb))
		f2fs_quota_off_umount(sbi->sb);
#endif
	f2fs_sync_inode_meta(sbi);
	/*
	 * Some dirty meta pages can be produced by recover_orphan_inodes()
	 * failed by EIO. Then, iput(node_inode) can trigger balance_fs_bg()
	 * followed by write_checkpoint() through f2fs_write_node_pages(), which
	 * falls into an infinite loop in sync_meta_pages().
	 */
	truncate_inode_pages_final(META_MAPPING(sbi));
#ifdef CONFIG_QUOTA
free_sysfs:
#endif
	f2fs_unregister_sysfs(sbi);
free_root_inode:
	dput(sb->s_root);
	sb->s_root = NULL;
free_stats:
	f2fs_destroy_stats(sbi);
free_node_inode:
	release_ino_entry(sbi, true);
	truncate_inode_pages_final(NODE_MAPPING(sbi));
	iput(sbi->node_inode);
free_nm:
	destroy_node_manager(sbi);
free_sm:
	destroy_segment_manager(sbi);
free_devices:
	destroy_device_list(sbi);
	kfree(sbi->ckpt);
free_meta_inode:
	make_bad_inode(sbi->meta_inode);
	iput(sbi->meta_inode);
free_io_dummy:
	mempool_destroy(sbi->write_io_dummy);
free_percpu:
	destroy_percpu_info(sbi);
free_bio_info:
	for (i = 0; i < NR_PAGE_TYPE; i++)
		kfree(sbi->write_io[i]);
free_options:
#ifdef CONFIG_QUOTA
	for (i = 0; i < MAXQUOTAS; i++)
		kfree(sbi->s_qf_names[i]);
#endif
	kfree(options);
free_sb_buf:
	kfree(raw_super);
free_sbi:
	if (sbi->s_chksum_driver)
		crypto_free_shash(sbi->s_chksum_driver);
	kfree(sbi);

	/* give only one another chance */
	if (retry) {
		retry = false;
		shrink_dcache_sb(sb);
		goto try_onemore;
	}
	return err;
}
```
