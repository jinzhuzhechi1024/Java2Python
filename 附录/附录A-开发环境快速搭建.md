# 附录A 开发环境快速搭建

## A.1 Python 3.12安装

### macOS
```bash
brew install python@3.12
python3.12 --version
```

### Linux (Ubuntu)
```bash
sudo apt install python3.12 python3.12-venv
```

### Windows
从 [python.org](https://www.python.org/downloads/) 下载安装包，
安装时勾选"Add to PATH"。

---

## A.2 虚拟环境

```bash
# 创建虚拟环境
python3.12 -m venv .venv

# 激活
source .venv/bin/activate      # macOS/Linux
.venv\Scripts\activate        # Windows

# 退出
deactivate
```

---

## A.3 包管理

```bash
# pip基础
pip install pandas
pip install -r requirements.txt
pip freeze > requirements.txt

# poetry（推荐现代项目）
pip install poetry
poetry init
poetry add fastapi uvicorn
poetry install
```

---

## A.4 IDE配置

### PyCharm
1. 下载 [PyCharm Community](https://www.jetbrains.com/pycharm/)
2. 打开项目→设置Python Interpreter为.venv
3. 安装插件：mypy、pytest

### Jupyter Notebook
```bash
pip install jupyter
jupyter notebook  # 或 jupyter lab
```

### VS Code
安装扩展：Python、Pylance、Jupyter、Ruff

---

## A.5 静态检查

```bash
pip install mypy pyright
mypy --strict src/
pyright src/
```
