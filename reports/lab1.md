# Lab 1 Utilities

## 实验目的

1. 运行xv6系统
2. 通过实现`sleep`，`pingpong`，和`find`等函数，体会系统调用

## 实验步骤

### Boot xv6

这部分稍作改动，为的是Boot的同时，将代码传到GitHub上托管

```bash
git clone git://g.csail.mit.edu/xv6-labs-2020 my_xv6_lab # clone到指定文件夹中
cd my_xv6_lab
git remote set-url origin git@github.com:Baokker/my_xv6_lab.git # 修改origin源
# 其实后来想应该换成add的，例如
# git remote add origin git@...
# 这样子更合理
git push # push到GitHub，安全验证可以通过SSH-key实现，此处不赘述
make qemu # 运行
# ctrl+a x 退出QEMU
```

### sleep

代码编辑器采用的是VS code

此实验难度不大，参照其他代码即可，只需要调用一个sleep的系统调用：

（说明：采用`git diff`显示代码差异，易于观看）

```c
+#include "kernel/types.h"
+#include "kernel/stat.h"
+#include "user/user.h"
+
+int main(int argc, char *argv[]){
+    if (argc < 2){
+        fprintf(2,"Error: Lacking an argument...\n");
+        exit(1);
+    }
+
+    sleep(atoi(argv[1]));
+
+    exit(0);
+}
```

记得在Makefile里将编写的utils加入到`UPROGS`中。下面的代码为完成所有Lab1后Makefile的更改。

```c
@@ -149,6 +149,11 @@ UPROGS=\
 	$U/_grind\
 	$U/_wc\
 	$U/_zombie\
+	$U/_sleep\
+	$U/_pingpong\
+	$U/_find\
+	$U/_xargs\
+	$U/_primes\
```

### pingpong

```c
+#include "kernel/types.h"
+#include "kernel/stat.h"
+#include "user/user.h"
+
+int main(int argc, char *argv[]){
+    if (argc != 1){
+        fprintf(2,"Error: No need for arguments...\n");
+        exit(1);
+    }
+
+    int p[2];
+    pipe(p);
+
+    if (fork() == 0){// child
+        close(p[0]); // close write
+
+        char temp = 'x';
+        if (write(p[1],&temp,1))
+            fprintf(0,"%d: received ping\n",getpid());    
+
+        close(p[1]);
+    }
+    else{
+        wait((int *)0);
+        close(p[1]); // close read
+
+        char temp;
+        if (read(p[0],&temp,1))
+            fprintf(0,"%d: received pong\n",getpid());    
+
+        close(p[0]);    
+    }
+
+    exit(0);
+}
```

### primes

```c
+#include "kernel/types.h"
+#include "kernel/stat.h"
+#include "user/user.h"
+
+void primes(int *input, int count)
+{
+  if (count == 0) {
+    return;
+  }
+
+  int p[2], i = 0, prime = *input;
+  pipe(p);
+  char buff[4];
+  printf("prime %d\n", prime);
+  if (fork() == 0) {
+	close(p[0]);
+	for (; i < count; i++) {
+	  write(p[1], (char *)(input + i), 4);
+	}
+	close(p[1]);
+	exit(0);
+  } else {
+	close(p[1]);
+	count = 0;
+	while (read(p[0], buff, 4) != 0) {
+	  int temp = *((int *)buff);
+	  if (temp % prime) {
+	    *input++ = temp;
+		  count++;
+	  }
+	}
+	primes(input - count, count);
+	close(p[0]);
+	wait(0);
+  }
+}
+
+int main(int argc, char *argv[]) {
+  int input[34], i = 0;
+  for (; i < 34; i++) {
+    input[i] = i + 2;
+  }
+  primes(input, 34);
+  exit(0);
+}
```



### find

这部分主要是参考了ls.c中的内容

仍然需要牢记，strcmp在两个char*相同时返回**0**！

```c
+#include "kernel/types.h"
+#include "kernel/stat.h"
+#include "user/user.h"
+#include "kernel/fs.h"
+
+// strcmp return 0 if two char* is equal !!
+
+void find(char *path, char *filename)
+{
+  char buf[512], *p;
+  int fd;
+  struct dirent de;
+  struct stat st;
+
+  if ((fd = open(path, 0)) < 0)
+  {
+    fprintf(2, "find: cannot open %s\n", path);
+    return;
+  }
+
+  if (fstat(fd, &st) < 0)
+  {
+    fprintf(2, "find: cannot stat %s\n", path);
+    close(fd);
+    return;
+  }
+
+  if (st.type == T_FILE)
+  { // it should be a dir
+    fprintf(2, "find: can't find files in a file\n");
+    exit(1);
+  }
+
+  // T_DIR:
+  if (strlen(path) + 1 + DIRSIZ + 1 > sizeof buf)
+  { // too long ..
+    printf("find: path too long\n");
+    exit(1);
+  }
+
+  strcpy(buf, path);
+
+  p = buf + strlen(buf);
+  *p++ = '/'; // p is pointed to the last file/dir name
+
+  while (read(fd, &de, sizeof(de)) == sizeof(de))
+  {
+    if (de.inum == 0)
+      continue;
+    memmove(p, de.name, DIRSIZ);
+    p[DIRSIZ] = 0;
+    if (stat(buf, &st) < 0)
+    {
+      fprintf(2, "find: cannot stat %s\n", buf);
+      continue;
+    }
+    // return 0 if s1 equals to s2 !!
+    if (st.type == T_DIR && strcmp(".", p) != 0 && strcmp("..", p) != 0)
+      find(buf, filename); // recursion
+    else if (st.type == T_FILE && strcmp(p, filename) == 0)
+      printf("%s\n", buf); // print the output
+  }
+
+  close(fd);
+}
+
+int main(int argc, char *argv[])
+{
+  if (argc < 2)
+  {
+    fprintf(2, "too little arguments..\n");
+    exit(1);
+  }
+  find(argv[1], argv[2]);
+  exit(0);
+}
```

### xargs

```c
+#include "kernel/types.h"
+#include "user/user.h"
+#include "kernel/param.h"
+
+int main(int argc, char *argv[]) {
+  int i, count = 0, k, m = 0;
+  char* line_split[MAXARG], *p;
+  char block[32], buf[32];
+  p = buf;
+  for (i = 1; i < argc; i++) {
+	line_split[count++] = argv[i];
+  }
+  while ((k = read(0, block, sizeof(block))) > 0) {
+    for (i = 0; i < k; i++) {
+	  if (block[i] == '\n') {
+		buf[m] = 0;
+		line_split[count++] = p;
+		line_split[count] = 0;
+		m = 0;
+		p = buf;
+		count = argc - 1;
+		if (fork() == 0) {
+		  exec(argv[1], line_split);
+		}
+		wait(0);
+	  } else if (block[i] == ' ') {
+		buf[m++] = 0;
+		line_split[count++] = p;
+		p = &buf[m];
+	  } else {
+		buf[m++] = block[i];
+	  }
+	}
+  }
+  exit(0);
+}
```

## 实验中遇到的问题及解决办法

- `primes`逻辑略难理解
- 写find的时候遇到的最大问题，并非在于代码本身的逻辑，而是记错了`strcmp`的用法。若比较的两字符串相同，则该函数返回**0**，然而我一直写成了非0，导致卡了很久...
- xv6的命名（或者说很多用c写的算法/数据结构）都比较抽象，加大理解难度。例如fmtname光看变量名也不知道是什么（查阅后得知，原来是format name的意思），揣测了源码后才知道，原来是把绝对路径后的文件或文件夹名提取出来

## 实验心得

平时各种用户程序的执行，本质上都离不开对系统调用的合理运用。这需要系统调用一方面提供足够全面的功能，另一方面需要对有意或无意对操作系统产生破坏的程序进行预防。

![image-20220731100624392](img\image-20220731100624392.png)