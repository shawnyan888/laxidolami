# AGENTS.md

数据处理脚本集合，Python 3.11，无 Web 框架。

## 命令

```bash
# 运行
python main.py --input data/raw.csv --output data/out.parquet

# 测试
pytest tests/ -v
```

## 约定

- 数据处理函数放 `pipeline/`，输入输出均为 `pandas.DataFrame`，禁止在函数内打印。
- 新增脚本必须在 `tests/` 中有对应测试文件，命名 `test_<模块名>.py`。
- 处理大文件时用 `chunksize` 分块读取，单块上限 100 万行。
- 错误信息与日志用英文，包含输入文件名和行号。
