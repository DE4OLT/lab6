import os
import json
import logging
from functools import wraps


class FileError(Exception):
    pass


class FileNotFound(FileError):
    pass


class FileCorrupted(FileError):
    pass


def logged(exc_type, mode: str = "console"):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            logger = logging.getLogger("file_logger")
            logger.setLevel(logging.ERROR)

            if not logger.handlers:
                if mode == "console":
                    handler = logging.StreamHandler()
                elif mode == "file":
                    handler = logging.FileHandler("file_operations.log", encoding="utf-8")
                else:
                    raise ValueError("Unknown logging mode")

                formatter = logging.Formatter("%(asctime)s - %(levelname)s - %(message)s")
                handler.setFormatter(formatter)
                logger.addHandler(handler)

            try:
                return func(*args, **kwargs)
            except exc_type as e:
                logger.error("Помилка у методі %s: %s", func.__name__, str(e))
                raise
        return wrapper
    return decorator


class JsonFileManager:
    def __init__(self, filepath: str):
        self.filepath = filepath

        if not os.path.exists(self.filepath):
            raise FileNotFound(f"Файл '{self.filepath}' не знайдено")

        try:
            with open(self.filepath, "r", encoding="utf-8") as f:
                content = f.read().strip()
                if content == "":
                    self._init_empty_json()
                else:
                    f.seek(0)
                    json.load(f)
        except Exception as e:
            raise FileCorrupted(f"Файл '{self.filepath}' пошкоджено: {e}")

    @staticmethod
    @logged(FileError, mode="file")
    def create_file(filepath: str, initial_data=None):
        if initial_data is None:
            initial_data = []

        try:
            with open(filepath, "w", encoding="utf-8") as f:
                json.dump(initial_data, f, ensure_ascii=False, indent=2)
        except Exception as e:
            raise FileCorrupted(f"Неможливо створити файл '{filepath}': {e}")

        return JsonFileManager(filepath)

    def _init_empty_json(self):
        with open(self.filepath, "w", encoding="utf-8") as f:
            json.dump([], f, ensure_ascii=False, indent=2)

    @logged(FileError, mode="console")
    def read(self):
        try:
            with open(self.filepath, "r", encoding="utf-8") as f:
                return json.load(f)
        except FileNotFoundError:
            raise FileNotFound(f"Файл '{self.filepath}' не знайдено")
        except Exception as e:
            raise FileCorrupted(f"Помилка читання файлу '{self.filepath}': {e}")

    @logged(FileError, mode="file")
    def write(self, data):
        try:
            # Сортування від Я → А
            if isinstance(data, list):
                data = sorted(data, key=lambda x: x["name"], reverse=True)

            with open(self.filepath, "w", encoding="utf-8") as f:
                json.dump(data, f, ensure_ascii=False, indent=2)

        except Exception as e:
            raise FileCorrupted(f"Помилка запису у файл '{self.filepath}': {e}")

    @logged(FileError, mode="file")
    def append(self, item):
        try:
            data = self.read()

            if not isinstance(data, list):
                raise FileCorrupted("JSON має бути списком для append()")

            data.append(item)

            # Сортування після додавання
            data = sorted(data, key=lambda x: x["name"], reverse=True)

            self.write(data)

        except FileError:
            raise


if __name__ == "__main__":
    path = "data.json"

    if not os.path.exists(path):
        manager = JsonFileManager.create_file(path, initial_data=[])
    else:
        manager = JsonFileManager(path)

    # Початковий список імен
    manager.write([
        {"name": "Роман"},
        {"name": "Василь"},
        {"name": "Андрій"},
        {"name": "Дмитро"}  
    ])

    print(manager.read())
