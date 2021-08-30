---
title: 监控进程CPU利用率
date: 2021-08-30 20:56:28
tags:
---

## 思路

一般可以使用 top 命令实时监控进程的CPU利用率,如进程共使用了四个CPU核,则 %CPU 这一列可能显示 400%

针对长时间运行的程序,想要评估程序的并发程度是否符合预期,可以编写脚本来更详细的监测进程的CPU利用率

这里主要通过读取 **/proc/PID/stat** 文件和 **/proc/stat** 系统文件,分别获取一段时间内进程所消耗的CPU时间和系统所消耗的总CPU时间,计算比值从而得到实时的CPU利用率

## 代码

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
pair<size_t, size_t> cpu_utilization_by_process(int pid);
vector< string > split_str(const std::string& str, char delim=' ', bool skip_empty=true);
bool collect_memory(int root_pid, size_t& total_memory);

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

    string cmd("cat /proc/cpuinfo| grep 'processor'| sort|uniq | wc -l");
    string res;
    exec_shell(cmd.c_str(), res);
    int cores = std::stoi(res);

    pair<size_t, size_t> last_cpu(0, 0);
    float cpu_utilization = 0;
    while (true)
    {
        auto curr_cpu = cpu_utilization_by_process(root_pid);
        if (curr_cpu.first == 0 && curr_cpu.second == 0)
            break;
        std::chrono::system_clock::time_point now = std::chrono::system_clock::now();
        std::time_t now_c = std::chrono::system_clock::to_time_t(now);
        if (last_cpu.first != 0)
            cpu_utilization = (curr_cpu.first - last_cpu.first)*100.0*cores/
            (curr_cpu.second - last_cpu.second);
        last_cpu = curr_cpu;
        cout<< std::put_time(std::localtime(&now_c), "%F %T")<<" "<<cpu_utilization<<endl;
        sleep(interval);
    }

    return 0;
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

pair<size_t, size_t> cpu_utilization_by_process(int pid)
{
    pair<size_t, size_t> result(0, 0);

#ifndef _WIN32
    size_t process_time = 0;
    string file_name = "/proc/"+to_string(pid)+"/stat";
    FILE* file = fopen(file_name.c_str(), "r");
    if (!file) return result;
    char  line[1024];
    if (fgets(line, 1024, file) != nullptr)
    {
        string sline(line);
        auto tmp = split_str(line);
        process_time += std::stoull(tmp[13]) + std::stoull(tmp[14]) +
            std::stoull(tmp[15]) + std::stoull(tmp[16]);
    }
    fclose(file);

    size_t cpu_time = 0;
    file_name = "/proc/stat";
    file = fopen(file_name.c_str(), "r");
    if (!file) return result;
    char  line2[1024];
    if (fgets(line2, 1024, file) != nullptr)
    {
        string sline(line2);
        auto tmp = split_str(line2);
        for (int i = 1; i <= 9; ++i)
            cpu_time += std::stoull(tmp[i]);
    }

    result.first = process_time;
    result.second = cpu_time;

#endif
    //printf("%ull %ull\n", result.first, result.second);

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

使用 gcc 编译代码 `g++ -std=c++17 monitor_process.cpp -o process_cpu`

首先运行待检测的程序,然后通过命令如 top 确定对应进程ID, 运行命令如 `./process_cpu 25201 1 > log` 监控 id 为25201的进程,刷新间隔为1秒,并将结果输出到磁盘文件

结果输出示例:

```
2021-08-30 20:54:07 0
2021-08-30 20:54:10 131.933
2021-08-30 20:54:13 37.6611
2021-08-30 20:54:16 143.261
2021-08-30 20:54:19 34.3109
```

注意:当进程结束,则监控程序也会退出

关于结果展示,直接将输出结果的第二列和第三列拷贝到 Excel 中,插入折线图即可看到CPU利用率随时间变化的情况