---
{"dg-publish":true,"permalink":"/01-note//busybox/syslogd/busyboxsyslog/","tags":"gardenEntry"}
---

# busybox-syslog修改帮助信息
> 本文使用的busybox版本：1.22.1

busybox syslogd的帮助信息生成于 `include\usage.h`，但不能直接修改 `usage.h` 文件，原因在 `usage.h` 开头也有说明：

```c
/* DO NOT EDIT. This file is generated from usage.src.h */
/* vi: set sw=8 ts=8: */
/*
 * This file suffers from chronically incorrect tabification
 * of messages. Before editing this file:
 * 1. Switch you editor to 8-space tab mode.
 * 2. Do not use \t in messages, use real tab character.
 * 3. Start each source line with message as follows:
 *    |<7 spaces>"text with tabs"....
 * or
 *    |<5 spaces>"\ntext with tabs"....
 */

·····

#define syslogd_full_usage "\n\n" \
       "System logging utility\n" \
	IF_NOT_FEATURE_SYSLOGD_CFG( \
       "(this version of syslogd ignores /etc/syslog.conf)\n" \
	) \
     "\n	-n		Run in foreground" \
     "\n	-O FILE		Log to FILE (default:/var/log/messages)" \
     "\n	-l N		Log only messages more urgent than prio N (1-8)" \
     "\n	-S		Smaller output" \
	IF_FEATURE_ROTATE_LOGFILE( \
     "\n	-s SIZE		Max size (KB) before rotation (default:200KB, 0=off)" \
     "\n	-b N		N rotated logs to keep (default:1, max=99, 0=purge)" \
     "\n	-F FILE		Log to FLASH FILE (default:/var/app/syslog/messages)" \
     "\n	-N FLASHSIZE 	Max size (KB) before rotation (default:512KB, 0=off)" \
	) \
	IF_FEATURE_REMOTE_LOG( \
     "\n	-R HOST[:PORT]	Log to HOST:PORT (default PORT:514)" \
     "\n	-L		Log locally and via network (default is network only if -R)" \
	) \
	IF_FEATURE_SYSLOGD_DUP( \
     "\n	-D		Drop duplicates" \
	) \
	IF_FEATURE_IPC_SYSLOG( \
     "\n	-C[size_kb]	Log to shared mem buffer (use logread to read it)" \
	) \
	IF_FEATURE_SYSLOGD_CFG( \
     "\n	-f FILE		Use FILE as config (default:/etc/syslog.conf)" \
	) \
	IF_FEATURE_KMSG_SYSLOG( \
     "\n	-K		Log to kernel printk buffer (use dmesg to read it)" \
	) \

······

```

可以得知 `usage.h` 是在编译过程中根据 `usage.src.h` 自动生成的，根据说明跳转到 `include\usage.src.h`，发现只有短短几行：

```c
/* vi: set sw=8 ts=8: */
/*
 * This file suffers from chronically incorrect tabification
 * of messages. Before editing this file:
 * 1. Switch you editor to 8-space tab mode.
 * 2. Do not use \t in messages, use real tab character.
 * 3. Start each source line with message as follows:
 *    |<7 spaces>"text with tabs"....
 * or
 *    |<5 spaces>"\ntext with tabs"....
 */
#ifndef BB_USAGE_H
#define BB_USAGE_H 1

#define NOUSAGE_STR "\b"

INSERT

#define busybox_notes_usage \
       "Hello world!\n"

#endif

```

这个 INSERT 是什么？
INSERT 将在编译时被脚本 `scripts/gen_build_files.sh` 替换为该命令目录下 `*.c` 中以 `//usage:` 打头的内容。
例如，syslogd 这个工具的帮助信息就保存在 `/busybox-1.22.1/sysklogd/syslogd.c` 这个文件中，如下所示，在编译 syslogd 时`gen_build_files.sh` 就会把这部分开头的 "//usage:" 去掉保存到 `usage.h` 中。
所以如果要修改现有功能的帮助信息的话，直接修改该工具名.c文件下以 `//usage:` 打头的内容就可。

`/busybox-1.22.1/sysklogd/syslogd.c`:

```c
//usage:#define syslogd_trivial_usage
//usage:       "[OPTIONS]"
//usage:#define syslogd_full_usage "\n\n"
//usage:       "System logging utility\n"
//usage:	IF_NOT_FEATURE_SYSLOGD_CFG(
//usage:       "(this version of syslogd ignores /etc/syslog.conf)\n"
//usage:	)
//usage:     "\n	-n		Run in foreground"
//usage:     "\n	-O FILE		Log to FILE (default:/var/log/messages)"
//usage:     "\n	-l N		Log only messages more urgent than prio N (1-8)"
//usage:     "\n	-S		Smaller output"
//usage:	IF_FEATURE_ROTATE_LOGFILE(
//usage:     "\n	-s SIZE		Max size (KB) before rotation (default:200KB, 0=off)"
//usage:     "\n	-b N		N rotated logs to keep (default:1, max=99, 0=purge)"
//usage:	)
//usage:	IF_FEATURE_REMOTE_LOG(
//usage:     "\n	-R HOST[:PORT]	Log to HOST:PORT (default PORT:514)"
//usage:     "\n	-L		Log locally and via network (default is network only if -R)"
//usage:	)
//usage:	IF_FEATURE_SYSLOGD_DUP(
//usage:     "\n	-D		Drop duplicates"
//usage:	)
//usage:	IF_FEATURE_IPC_SYSLOG(
/* NB: -Csize shouldn't have space (because size is optional) */
//usage:     "\n	-C[size_kb]	Log to shared mem buffer (use logread to read it)"
//usage:	)
//usage:	IF_FEATURE_SYSLOGD_CFG(
//usage:     "\n	-f FILE		Use FILE as config (default:/etc/syslog.conf)"
//usage:	)
/* //usage:  "\n	-m MIN		Minutes between MARK lines (default:20, 0=off)" */
//usage:	IF_FEATURE_KMSG_SYSLOG(
//usage:     "\n	-K		Log to kernel printk buffer (use dmesg to read it)"
//usage:	)
//usage:
//usage:#define syslogd_example_usage
//usage:       "$ syslogd -R masterlog:514\n"
//usage:       "$ syslogd -R 192.168.1.1:601\n"
```

下面对 `gen_build_files.sh` 进行分析，看它如何实现将 syslogd.c 开头的 "//usage:" 去掉，并保存到 `usage.h` 中

`/busybox-1.22.1/scripts/gen_build_files.sh`:

```shell
# (Re)generate include/usage.h
# We add line continuation backslash after each line,
# and insert empty line before each line which doesn't start
# with space or tab
sed -n -e 's@^//usage:\([ \t].*\)$@\1 \\@p' -e 's@^//usage:\([^ \t].*\)$@\n\1 \\@p' \
        "$srctree"/*/*.c "$srctree"/*/*/*.c \
| generate \
        "$srctree/include/usage.src.h" \
        "include/usage.h" \
        "/* DO NOT EDIT. This file is generated from usage.src.h */"
```

Linux `sed` 命令是利用脚本来处理文本文件。

`sed` 可依照脚本的指令来处理、编辑文本文件。

- -e 选项允许在同一行里执行多条命令,命令的执行顺序对结果有影响。如果两个命令都是替换命令，那么第一个替换命令将影响第二个替换命令的结果。

```shell
-e 's@^//usage:\([ \t].*\)$@\1 \\@p' -e 's@^//usage:\([^ \t].*\)$@\n\1 \\@p'
```

- `s`:替换

- `^`:字符串开头

- `$`:字符串结尾

- `@xxxxx@`:xxxxx代表正则表达式？？

- `\(  \)`表示将其中的内容看做一个分组，看做一个整体

- `[  ]` 表示匹配指定范围内的任意单个字符

- `[^  ]`表示匹配指定范围外的任意单个字符。

- `\t`:匹配一个制表符

- `[ \t]`: 表示空格和 `\t` 都能被匹配到

- `*` 表示前面的字符连续出现任意次，包括0次。

- `.` 表示表示匹配任意单个字符（换行符除外）。

- `.*` 表示任意长度的任意字符，与通配符中的 `*` 的意思相同。

- `\1`：表示直到最后一行???或者表示引用整个正则中第1个分组中的正则所匹配到的结果？？？

- `p`: -n 选项会禁止 sed 输出，但 p 标记会输出修改过的行，将二者匹配使用的效果就是只输出被替换命令修改过的行

- `srctree` 表示当前代码所在目录的路径。

  `"$srctree"/*/*.c "$srctree"/*/*/*.c` 表示 `/busybox-1.22.1/` 下所有`.c`文件
  
- `generate`：生成

正则表达式相关请看[正则表达式](正则表达式.md)

> 参考文章：https://blog.csdn.net/Kami_Jiang/article/details/121106478