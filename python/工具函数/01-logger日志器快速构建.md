```python
import os
import logging

def get_logger(name, level=None):
    """
    创建并返回一个带有特定格式和日志等级的日志记录器（logger）对象

    Args:
        name (str): logger的名字.
        level (int): The logging level (default: logging.INFO).

    Returns:
        logging.Logger: A configured logger instance.
    """

    if level is None:
        level = int(os.environ.get("LOG_LEVEL", logging.INFO))

    logger = logging.getLogger(name)
    logger.setLevel(level)

    # 避免重复添加 handler
    if not logger.handlers:
        # 创建一个控制台输出的 handler，并设置设置日志等级
        console_handler = logging.StreamHandler()
        console_handler.setLevel(level)

        # 创建日志格式器 时间-日志等级-文件名:行号-日志内容
        formatter = logging.Formatter(
            "%(asctime)s - %(levelname)s - %(filename)s:%(lineno)d - %(message)s"
        )
        console_handler.setFormatter(formatter)

        # 把 handler 加到 logger 上
        logger.addHandler(console_handler)

    return logger
```