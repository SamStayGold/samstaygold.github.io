---
title: jupyter lab打开超大文件后反复卡死问题修复
date: 2024-07-26 15:59 +0800
categories: [大数据, jupyter]
tags: [jupyter]
---

# 背景
为了支持算法用户 python 脚本的开发需求，我们线上有个 Jupyter Hub 服务器，Spawn 出 Jupyter Lab 的 server 端给用户使用。

有时用户会打开非常大的 excel 或者 csv 文件，就很有可能导致 Jupyter Lab 前端卡死。

# 问题分析
前端尝试打开超大csv文件时，页面崩溃重新加载，然后又崩溃，反复循环。

# 解决方案
## 简单方案
提醒用户在 csv 内容还未尝试加载时，抢先关闭掉对应 tab 页即可。手速够快，就可以修复。

## 复杂方案
先让用户进入 hub 控制面板，选择 stop my server，关闭掉对应用户的 jupyter server。
进入Jupyter Lab 部署的 linux 服务器，找到对应用户的 jupyter 配置路径，比如对于用户 sam，配置路径可能是
```
/data/home/sam/.jupyter/lab/workspaces
```
找到最近的jupyterlab-workspace json 文件，可以下载下来，在本地进行编辑。

举个栗子：
```
{
    "data": {
        "layout-restorer:data": {
            "main": {
                "dock": {
                    "type": "tab-area",
                    "currentIndex": 2,
                    "widgets": [
                        "editor:atlas_report_cost/main.py",
                        "editor:atlas_report_cost/config.py",
                        "editor:atlas_report_cost/report_cost_calculator.py",
                        "spreadsheet-editor:atlas_report_cost/csv_results/report_cost_2024-11-03.csv",
                        "spreadsheet-editor:atlas_report_cost/csv_results/table_cost_2024-10-25.csv",
                        "spreadsheet-editor:atlas_report_cost/csv_results/report_related_table_cost_2024-11-03.csv",
                        "spreadsheet-editor:atlas_report_cost/csv_results/table_cost_2024-11-03.csv",
                        "spreadsheet-editor:atlas_report_cost/csv_results/report_related_table_cost_2024-10-28.csv"
                    ]
                },
                "current": "editor:atlas_report_cost/report_cost_calculator.py"
            },
            "down": {
                "size": 0,
                "widgets": []
            },
            "left": {
                "collapsed": false,
                "current": "filebrowser",
                "widgets": [
                    "filebrowser",
                    "running-sessions",
                    "@jupyterlab/toc:plugin"
                ]
            },
            "right": {
                "collapsed": true,
                "widgets": [
                    "jp-property-inspector"
                ]
            },
            "relativeSizes": [
                0.20829875518672203,
                0.791701244813278,
                0
            ]
        },
        "file-browser-filebrowser:cwd": {
            "path": "atlas_report_cost"
        },
        "editor:atlas_report_cost/main.py": {
            "data": {
                "path": "atlas_report_cost/main.py",
                "factory": "Editor"
            }
        },
        "spreadsheet-editor:atlas_report_cost/csv_results/report_related_table_cost_2024-10-28.csv": {
            "data": {
                "path": "atlas_report_cost/csv_results/report_related_table_cost_2024-10-28.csv",
                "factory": "Spreadsheet Editor"
            }
        },
        "editor:atlas_report_cost/report_cost_calculator.py": {
            "data": {
                "path": "atlas_report_cost/report_cost_calculator.py",
                "factory": "Editor"
            }
        },
        "spreadsheet-editor:atlas_report_cost/csv_results/report_cost_2024-11-03.csv": {
            "data": {
                "path": "atlas_report_cost/csv_results/report_cost_2024-11-03.csv",
                "factory": "Spreadsheet Editor"
            }
        },
        "spreadsheet-editor:atlas_report_cost/csv_results/table_cost_2024-10-25.csv": {
            "data": {
                "path": "atlas_report_cost/csv_results/table_cost_2024-10-25.csv",
                "factory": "Spreadsheet Editor"
            }
        },
        "spreadsheet-editor:atlas_report_cost/csv_results/report_related_table_cost_2024-11-03.csv": {
            "data": {
                "path": "atlas_report_cost/csv_results/report_related_table_cost_2024-11-03.csv",
                "factory": "Spreadsheet Editor"
            }
        },
        "spreadsheet-editor:atlas_report_cost/csv_results/table_cost_2024-11-03.csv": {
            "data": {
                "path": "atlas_report_cost/csv_results/table_cost_2024-11-03.csv",
                "factory": "Spreadsheet Editor"
            }
        },
        "editor:atlas_report_cost/config.py": {
            "data": {
                "path": "atlas_report_cost/config.py",
                "factory": "Editor"
            }
        }
    },
    "metadata": {
        "id": "default"
    }
```

这里删除掉问题 csv 的类目，比如，"spreadsheet-editor:atlas_report_cost/csv_results/report_related_table_cost_2024-10-28.csv"

然后上传到服务器，覆盖掉原来的文件，重新启动对应用户的 jupyter server 即可。