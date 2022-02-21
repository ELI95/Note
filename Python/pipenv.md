- 安装 pipenv
```bash
pipx install pipenv
```

- 指定Python的版本创建虚拟环境
```bash
pipenv --python 3
```

- 根据Pipfile创建虚拟环境
```bash
pipenv install
```

- 安装包至当前项目
```bash
# 高度推荐用双引号包裹包版本标识(如 "requests>2.19")，这是为了避免在基于Unix的操作系统中出现 输入输出重定向问题
pipenv install <pkg>
```

- 根据requirements.txt创建虚拟环境
```bash
pipenv install -r path/to/requirements.txt
```
  
- 导出当前项目的requirements.txt
```bash
pipenv run pip freeze > path/to/requirements.txt
```

- 创建Pipfile.lock
```bash
pipenv lock
```

- 升级所有依赖包 
```bash
pipenv update
```

- 升级某个依赖包
```bash
pipenv update <pkg>
```

- 启动pipenv终端
```bash
pipenv shell
```

- 运行命令
```bash
pipenv run <command>
```

- 删除虚拟环境
```bash
pipenv --rm
```
