# 一、实验简介

> 《C 语言实现 Linux Shell 命令解释器》项目可以培养 Linux 系统编程能力，尤其是在多进程方面。可以了解fork、execvp 等重要的系统调用。另外还能深入到底层理解 Linux Shell 的功能的实现手段。为了测试方便，代码只放在一个文件中。

## 1.1 知识点
- Shell 的基本概念
- 进程控制相关的系统调用的使用（如 fork,exev）
- 信号的概念及系统调用 signal的使用

## 1.2 效果截图
![这里写图片描述](http://img.blog.csdn.net/20171016141912089?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvel9tbDExOA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
## 1.3 设计流程
![这里写图片描述](http://img.blog.csdn.net/20171016145603602?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvel9tbDExOA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
# 二、main 函数的设计
首先，以自顶向下的方式探讨一下在 Linux Shell 的周期内主要做了哪些事:
初始化：在这一步，一个典型的 Shell 应该读取配置文件并执行配置功能。这样可以改变 Shell 的行为。
解释：接下来，Shell 将从标准输入（可以是交互的方式或者一个脚本文件）中读入命令，然后执行它。
终止：在命令被执行之后，Shell 执行关闭命令，释放内存，最后终止。
详细见代码注释：
```
/* 主函数入口 */
/* 主函数入口 */
int main()
{
    char *command;
    int iter = 0;
    command = (char *)malloc(MAX+1); //用于存储命令语句

    chdir("/home/");  /* 通常在登陆 shell 的时候通过查找配置文件（/etc/passwd 文件）
                               中的起始目录，初始化工作目录，这里默认为家目录 */
    while(1)
    {
        iter = 0;
		//定义信号处理方式
		signal_set();
        //用于输出提示符
        print_prompt();  
        //扫描多条命令语句
        scan_command(command);    
        // 基于分号解析出单条命令语句，并以字符串的形式存进全局变量 all 中
        parse_semicolon(command);

        // 迭代执行单条命令语句
        while(all[iter] != NULL)
        {
            execute(all[iter]); //这是 shell 解释器的核心函数
            iter += 1;
        }
    }
}
```
- 注：print_prompt()、scan_command(command) 和 parse_semicolon(command) 三个函数是自定义函数，功能分别为：用于输出提示符，扫描多条命令语句，基于分号解析出单条命令语句，并以字符串的形式存进全局变量 all 中。概念简单，其详细代码在项目完整代码中。

# 三、程序的核心功能 execute 函数
execute函数完成的主要功能如下：
- 判断用户执行的是否是 Shell 的内建命令，如果是则执行（这里为了强调概念，没有过多的添加内建命令）
- 解析单条命令语句，即将命令名和命令参数解析并保存在字符串数组，以便调用 bf_exec命令。
- 识别输入输出重定向符号，利用底层系统调用函数 dup2完成上层输入输出重定向
- 识别后台运行符号，通过设置自定义函数 bf_exec的模式为1实现，这里要说明一点：所谓后台执行，无非就是让前台 Shell 程序以非阻塞的形式执行，并没有对后台程序做任何操作
- 这部分的完整代码如下：
```
/* 执行单条命令行语句 */
void execute(char *command)
{
    char *arg[MAX_COMM];
    char *try;
    arg[0] = parse(command, 0);        //获得命令名称的字符串指针，如“ls”
    int t = 1;
    arg[t] = NULL;   
    if (strcmp(arg[0], "cd") == 0)     // 处理内嵌命令“cd”的情况
    {
        try = parse(command, 1);
        cd(try);
        return ;
    }
    if (strcmp(arg[0], "exit") == 0) // 为了方便用户推出shell
        exit(0);

    // 循环检测剩下的命令参数，即检测是否：重定向？管道？后台执行？普通命令参数？
    while (1)
    {
        try = parse(command, 1);
        if (try == NULL)
            break;
        else if (strcmp(try, ">") == 0)  // 重定向到一个文件的情况
        {
            try = parse(command, 1);   // 得到输出文件名
            file_out(arg, try, 0);        // 参数 0 代表覆盖的方式重定向
            return;
        }
        else if (strcmp(try, ">>") == 0)   // 追加重定向到一个文件
        {
            try = parse(command, 1);
            file_out(arg, try, 1);        // 参数 1 代表追加的方式重定向
            return;
        }
        else if (strcmp(try, "<") == 0)    // 标准输入重定向
        {
            try = parse(command, 1);      // 输入重定向的输入文件
            char *out_file = parse(command, 1);    // 输出重定向的输出文件
            if (out_file != NULL)
            {
                if (strcmp(out_file, ">") == 0)   // 输入输出文件给定
                {
                    out_file = parse(command, 1);
                    if (out_file == NULL)
                    {
                        printf("Syntax error !!\n");
                        return;
                    }
                    else
                        file_in(arg, try, out_file, 1);     // 参数 1 针对双重定向 < >
                }
                else if (strcmp(out_file, ">>") == 0)
                {
                    out_file = parse(command, 1);
                    if (out_file == NULL)
                    {
                        printf("Syntax error !!\n");
                        return;
                    }
                    else
                        file_in(arg, try, out_file, 2);       // 参数 2 针对双重定向 < >>
                }
            }
            else
            {
                file_in(arg, try, NULL, 0);        // 模式 0 针对单一的输入重定向 
            }
        }
        //处理后台进程
        else if (strcmp(try, "&") == 0)   // 后台进程
        {
            bf_exec(arg, 1);     // bf_exec 的第二个参数为 1 代表后台进程
            return;
        }
        else       //try是一个命令参数
        {
            arg[t] = try;
            t += 1;
            arg[t] = NULL;
        }
    }

    bf_exec(arg, 0);     // 参数 0 表示前台运行
    return;
}
```
## 3.1处理标准输出重定向问题的函数： file_out
本函数的定义为： **void file_out(char \*arg[], char \*out_file, int type)。** arg 代表命令行参数。 **out_file** 代表重定向的文件名。 通过** dup2(f, 1)** 语句使得标准输出文件（文件描述符为1）重定向到制定的文件**out_file**(文件描述符f)。至于是追加重定向还是覆盖重定向取决于函数 open 的打开模式（是否是 **O_APPEND**）。
这部分的完整代码如下：
```
/* 处理输出重定向文件的问题 */
void file_out(char *arg[], char *out_file, int type)
{
    int f;
    current_out = dup(1);         
    if(type == 0)        //处理以覆盖的方式重定向“>”
    {
        f = open(out_file, O_WRONLY | O_CREAT, 0777); 
        dup2(f, 1); //复制文件描述符f，并指定为 1，也就是让标准输出重定向到指定的文件
        close(f);
        bf_exec(arg, 0);
    }
    else                 // 处理以追加的方式重定向“>>”
    {
        f = open(out_file, O_WRONLY | O_CREAT | O_APPEND , 0777); //以 O_APPEND 模式打开
        dup2(f, 1);
        close(f);
        bf_exec(arg, 0);
    }
}
```
## 3.2 处理标准输入重定向问题的函数： file_in
本函数的定义原型为 **void file_in(char \*arg[], char \*in_file, char \*out_file, int type)** 类似于 file_out函数，该本分仍然采用 dup2(in, 0)来完成标准输入文件（文件描述符0）重定向到执行的文件 in_file（文件描述符为 in）。
- 这部分的完整代码如下：
```
/* 处理输入重定向文件的问题 */
void file_in(char *arg[], char *in_file, char *out_file, int type)
{
    int in;
    in = open(in_file, O_RDONLY);
    current_in = dup(0); //
    dup2(in, 0);
    close(in);
    if(type == 0)    // 单一的输入重定向
    {
        printf("Going to execute bf_exec\n");          //debug remove it
        bf_exec(arg, 0);
    }
    else if(type == 1)            // 双重重定向 `< ... >` 
    {
        file_out(arg, out_file, 0);
    }
    else                         // 双重重定向 `< ... >>`
    {
        file_out(arg, out_file, 1);
    }
    return;
}
```

## 3.3 实现命令执行的最后一步： bf_exec
本函数定义的原型为：**void bf_exec(char \*arg[], int type)**, arg 指命令行，type标识前台还是后台运行。
- 实现命令的执行必须在 Shell 父进程下 fork 一个子进程，并在子进程中调用 execvp 函数装载子进程执行的代码（所要执行的命令）。
- 程序的前后台执行取决于父进程在子进程执行过程中是否是阻塞的。代码中，前台进程的父进程需要调用 wait(&pid) 阻塞，等待前台执行的命令执行完才能唤醒。
- 这部分的完整代码如下：
```
/* 创建子进程，调用 execvp 函数执行命令程序（前后台）*/
void bf_exec(char *arg[], int type)
{
    pid_t pid;
    if(type == 0)    // 前台执行
    {
        if((pid = fork()) < 0)
        {
            printf("*** ERROR: forking child process failed\n");
            return ;
        }
        // 父子进程执行代码的分离
        else if(pid == 0)    //子进程
        {
            signal(SIGTSTP, SIG_IGN);
            signal(SIGINT,SIG_DFL);//用于杀死子进程
			signal(SIGQUIT, SIG_IGN);
            execvp(arg[0], arg);  // execvp 用于在子进程中替换为另一段程序
        }
        else     //父进程
        {
			signal(SIGCHLD, bg_signal_handle);
			signal(SIGINT, sig_handle); // SIGINT = 2， 用户键入Ctrl-C
			signal(SIGQUIT, sig_handle); /* SIGQUIT = 3 用户键入Ctr+\ */
			signal(SIGTSTP, sig_handle);  // SIGTSTP =20，一般有Ctrl-Z产生
			pid_t c;
            c = wait(&pid);  //等待子进程结束
            dup2(current_out, 1); //还原默认的标准输出
            dup2(current_in, 0); //还原默认的标准输入
            return;
        }
    }
    else            // 后台执行
    {
        if((pid = fork()) < 0)
        {
            printf("*** ERROR: forking child process failed\n");
            return ;
        }
        else if(pid == 0)    // 子进程
        {
			signal(SIGINT, SIG_IGN); 
			signal(SIGQUIT, SIG_IGN); 
			signal(SIGTSTP,SIG_IGN);
            execvp(arg[0], arg);
        }
        else     // 父进程
        {
			signal(SIGCHLD, bg_signal_handle);
			signal(SIGINT, sig_handle); // SIGINT = 2， 用户键入Ctrl-C
			signal(SIGQUIT, sig_handle); /* SIGQUIT = 3 用户键入Ctr+\ */
			signal(SIGTSTP, sig_handle);  // SIGTSTP =20，一般有Ctrl-Z产生
            bg_struct_handle(pid, arg, 0);
            dup2(current_out, 1);
            dup2(current_in, 0);
            return ;
        }

    }
}
```
# 四、实验总结
通过本实验，我们完成了一个支持输入输出重定向，支持后台运行的 Linux Shell 命令解释器。通过该项目的学习，可以更进一步的了解 Linux Shell 的工作机制。在使用重定向 > 和>> 时 建议将目录切换到自己的家目录，否则没权限建立重定向文件。
但是在本次实验中出现了一点bug还没有解决，重定`< file1`和`< file1 >> file2` 的功能在运行时没有效果。希望各位童鞋能够改进代码，代码中的不足也希望大家能够指出，谢谢。
- 该项目的完整代码见:github

# 五、参考资料

- [实验楼](https://www.shiyanlou.com/)
- [UNIX环境高级编程》](https://book.douban.com/subject/1788421/)
