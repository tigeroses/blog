---
title: 监控进程内存
date: 2020-09-21 21:47:53
tags: linux cpp
---

## 缘由

需要获取某程序运行过程中的内存消耗,一般情况可以使用 `top` 命令来人工分析,不过我遇到一个程序其内部调用包括 python, R, 以及一系列 linux 命令,这就导致人工统计不太现实

问题变成统计进程及其子进程的内存使用,可以通过 `pstree` 命令查看进程与子进程的关系,但是其输出图形,不太方便获取所有子进程ID,因此打算自己编写 C++ 代码来实现

## 思路

用伪码表示:

```py
当待查询进程存在:
    遍历用户所有进程,获取每个进程的ID和父进程ID及内存
    维护一个表,记录与待查询进程相关的子进程及其内存,初始化只有待查询进程
    遍历所有进程:
        如果当前进程的父进程在表中:
            将此进程及对应内存加入表
    汇总表,得出总内存,并打印
```

这里细节是如何高效的更新表,可以将问题抽象为由一组边来构建树的过程,每个进程都有唯一的进程id(pid)和父进程id(ppid),正常来说一个系统所有的进程可以构建成一棵树(linux系统上所有进程都是由其他进程fork来的),不过我们只想查询某个用户下的进程,因此结果会构建成多棵树,只要遍历找到某个树的某个节点为感兴趣的进程id,以此节点作为根节点,遍历整棵树汇总内存即为结果

不过为了实现简单,我这里没有采用构建树的方式,而是直接遍历,遇到相关的进程就更新进表中,同时删除掉此进程;当某次遍历后维护进程的链表长度没有发生改变,说明所有子进程已查找完毕;这种计算方式对少量数据情况还是挺快的

## 代码

查询某个进程的信息比如内存占用,父进程ID等,linux 系统可以通过解析 `/proc/pid/status` 文件来获取

查找某用户所有进程,可使用命令 `ps -U username`

```cpp
// monitor_process.cpp
#include <iostream>
#include <string>
#include <vector>
#include <list>
#include <sstream>
#include <unordered_map>
#include <chrono>
#include <iomanip>

#include <ctime>

#include <sys/sysinfo.h>
#include <unistd.h>
#include <string.h>

using namespace std;

int exec_shell(const char* cmd, std::string & res);
pair<int, size_t> physical_memory_used_by_process(int pid);
vector< string > split_str(const std::string& str, char delim=' ', bool skip_empty=true);
bool collect_memory(int root_pid, size_t& total_memory);

const string username = "zhangsan";

struct ProcessInfo
{
    int pid;
    int ppid;
    size_t memory;
};

int main(int argc, char** argv)
{
    if (argc != 3)
    {
        cout<<"Enter <pid> <interval>\n";
        exit(-1);
    }
    int root_pid = stoi(argv[1]);
    int interval = stoi(argv[2]);

    while (true)
    {
        size_t memory = 0;
        if (!collect_memory(root_pid, memory))
            break;
        std::chrono::system_clock::time_point now = std::chrono::system_clock::now();
        std::time_t now_c = std::chrono::system_clock::to_time_t(now);
        cout<< std::put_time(std::localtime(&now_c), "%F %T")<<" "<<memory<<endl;
        sleep(interval);
    }

    return 0;
}

bool collect_memory(int root_pid, size_t& total_memory)
{
    string get_all_process = "ps -U " + username + " | awk '{if (NR>1) printf $1\" \"}'";
    string res;
    exec_shell(get_all_process.c_str(), res);
    vector<string> pids = split_str(res);
    list<ProcessInfo> process_infos;
    for (auto& pid : pids)
    {
        auto [ppid, memory] = physical_memory_used_by_process(stoi(pid));
        // cout<<"pid: "<<pid <<" ppid: "<<ppid<<" "<<memory<<endl;
        ProcessInfo pi;
        pi.pid = stoi(pid);
        pi.ppid = ppid;
        pi.memory = memory;
        process_infos.push_back(pi);
    }

    unordered_map<int, size_t> determined;
    // Init using the root process
    for (auto& pi : process_infos)
    {
        if (pi.pid == root_pid)
        {
            determined.insert({root_pid, pi.memory});
            break;
        }
    }
    if (determined.empty())
        return false;

    // Collect all childs process
    while (true)
    {
        auto iter = process_infos.begin();
        int old_size = process_infos.size();
        while (iter != process_infos.end())
        {
            //cout<<"skip pid: "<<iter->pid<<endl;
            //cout<<"determined size: "<<determined.size()<<endl;
            if (determined.count(iter->ppid) != 0)
            {
                determined[iter->pid] = iter->memory;
                //cout<<"add pid: "<<iter->pid<<" memory: "<<iter->memory<<endl;
                process_infos.erase(iter);
                break;
            }
            iter++;
        }
        int new_size = process_infos.size();
        if (old_size == new_size)
            break;
    }
    for (auto& [_, m] : determined)
        total_memory += m;

    return true;
}

int exec_shell(const char* cmd, std::string & res)
{
    FILE* pp = popen(cmd, "r");  // make pipe
    if (!pp)
    {
        return -1;
    }
    char tmp[1024];  // store the stdout per line
    while (fgets(tmp, sizeof(tmp), pp) != NULL)
    {
        if (tmp[strlen(tmp) - 1] == '\n')
        {
            tmp[strlen(tmp) - 1] = '\0';
        }
        res += tmp;
    }

    // close pipe, the return code is cmd's status
    // returns the exit status of the terminating command processor
    // -1 if an error occurs
    int rtn = pclose(pp);
#ifndef _WIN32
    rtn = WEXITSTATUS(rtn);
#endif

    return rtn;
}

pair<int, size_t> physical_memory_used_by_process(int pid)
{
    pair<int, size_t> result(-1, 0);
#ifndef _WIN32
    string file_name = "/proc/"+to_string(pid)+"/status";
    FILE* file = fopen(file_name.c_str(), "r");
    if (!file) return result;
    char  line[128];
    while (fgets(line, 128, file) != nullptr)
    {
        if (strncmp(line, "VmRSS:", 6) == 0)
        {
            int len = strlen(line);

            const char* p = line;
            for (; std::isdigit(*p) == false; ++p)
            {
            }

            line[len - 3] = 0;
            result.second        = atoi(p);
        }
        else if (strncmp(line, "PPid:", 5) == 0)
        {
            int len = strlen(line);

            const char* p = line;
            for (; std::isdigit(*p) == false; ++p)
            {
            }

            result.first        = atoi(p);
        }
    }
    fclose(file);
#endif

    return result;
}

vector< string > split_str(const std::string& str, char delim, bool skip_empty)
{
    std::istringstream iss(str);
    vector< string >   res;
    for (std::string item; getline(iss, item, delim);)
        if (skip_empty && item.empty())
            continue;
        else
            res.push_back(item);
    return res;
}
```

## 结果

使用 gcc9.1 编译代码 `g++ -std=c++17 monitor_process.cpp -o pm`

首先运行待检测的程序,然后通过命令如 `top` 确定对应进程ID, 运行命令如 `./pm 25201 2` 监控 id 为25201的进程,刷新间隔为2秒

结果输出:

```text
2020-09-18 17:37:05 1932
2020-09-18 17:37:07 1932
2020-09-18 17:37:09 1932
```

这里输出的内存单位是 KB

注意:由于需要进程启动之后才能开启监控,导致进程内存无法从0开始;当进程结束,则监控程序也会退出

关于结果展示,直接将输出结果的第二列和第三列拷贝到 Excel 中,插入折线图即可看到内存随时间变化情况