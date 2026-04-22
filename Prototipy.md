# ne-beyte-pojaluysta
Итак, по итогу было решено сделать простенькую программу по типу блокнота. В данном блакноте можно добавлять задачи, уберать задачи, делать очистку задачи, посмотреть задачу, отредактировать, посмотреть статистику выполненых и удалённых задач.
```shell
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

"""
Расширенный менеджер задач (To-Do блокнот)
Функции: добавление, просмотр, редактирование, удаление, отметка выполнения,
поиск, статистика, приоритеты, дедлайны, фильтрация, сортировка, экспорт/импорт,
напоминания, логирование, резервное копирование.
"""

import os
import sys
import json
import csv
import shutil
from datetime import datetime, date
from typing import List, Dict, Any, Optional, Tuple

# Константы
FILENAME = 'tasks.json'          # Новый формат JSON
BACKUP_DIR = 'task_backups'      # Папка для резервных копий
LOG_FILE = 'task_manager.log'    # Лог-файл
OLD_FILENAME = 'tasks.txt'       # Старый формат для миграции

# ANSI цвета для красивого вывода (если терминал поддерживает)
COLORS = {
    'reset': '\033[0m',
    'red': '\033[91m',
    'green': '\033[92m',
    'yellow': '\033[93m',
    'blue': '\033[94m',
    'magenta': '\033[95m',
    'cyan': '\033[96m',
    'bold': '\033[1m'
}

def supports_color() -> bool:
    """Проверяет, поддерживает ли терминал цветной вывод."""
    return hasattr(sys.stdout, 'isatty') and sys.stdout.isatty()

def colorize(text: str, color: str) -> str:
    """Оборачивает текст ANSI-цветом, если поддержка есть."""
    if supports_color() and color in COLORS:
        return f"{COLORS[color]}{text}{COLORS['reset']}"
    return text

def log_action(action: str) -> None:
    """Записывает действие в лог-файл с временной меткой."""
    timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    try:
        with open(LOG_FILE, 'a', encoding='utf-8') as f:
            f.write(f"[{timestamp}] {action}\n")
    except IOError:
        pass  # Не критично, если лог не пишется

def ensure_backup_dir() -> None:
    """Создаёт папку для резервных копий, если её нет."""
    if not os.path.exists(BACKUP_DIR):
        os.makedirs(BACKUP_DIR)
        log_action("Создана папка для резервных копий")

def create_backup(tasks: List[Dict[str, Any]]) -> str:
    """
    Создаёт резервную копию текущего списка задач.
    Возвращает путь к созданному файлу.
    """
    ensure_backup_dir()
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    backup_path = os.path.join(BACKUP_DIR, f"tasks_backup_{timestamp}.json")
    try:
        with open(backup_path, 'w', encoding='utf-8') as f:
            json.dump(tasks, f, ensure_ascii=False, indent=2)
        log_action(f"Создана резервная копия: {backup_path}")
        return backup_path
    except Exception as e:
        log_action(f"Ошибка создания резервной копии: {e}")
        return ""

def migrate_from_old_format() -> List[Dict[str, Any]]:
    """
    Миграция из старого текстового формата (tasks.txt) в новый JSON-формат.
    Старый формат: title|True/False
    Новый добавляет priority, created_at, due_date.
    """
    if not os.path.exists(OLD_FILENAME):
        return []
    tasks = []
    try:
        with open(OLD_FILENAME, 'r', encoding='utf-8') as f:
            lines = f.readlines()
        for line in lines:
            line = line.strip()
            if line and '|' in line:
                title, status = line.split('|', 1)
                done = (status == 'True')
                task = {
                    'title': title,
                    'done': done,
                    'priority': 'medium',      # значение по умолчанию
                    'created_at': datetime.now().isoformat(),
                    'due_date': None
                }
                tasks.append(task)
        if tasks:
            # Переименуем старый файл, чтобы не мешал
            os.rename(OLD_FILENAME, OLD_FILENAME + ".migrated")
            log_action(f"Миграция из {OLD_FILENAME}: {len(tasks)} задач")
        return tasks
    except Exception as e:
        log_action(f"Ошибка миграции: {e}")
        return []

def load_tasks() -> List[Dict[str, Any]]:
    """
    Загружает задачи из JSON-файла.
    Если файла нет, пытается выполнить миграцию из старого формата.
    Возвращает список задач.
    """
    tasks = []
    if os.path.exists(FILENAME):
        try:
            with open(FILENAME, 'r', encoding='utf-8') as f:
                tasks = json.load(f)
            log_action(f"Загружено {len(tasks)} задач из {FILENAME}")
        except (json.JSONDecodeError, IOError) as e:
            log_action(f"Ошибка загрузки JSON: {e}")
            print(colorize("Ошибка чтения файла задач. Будет создан новый список.", 'red'))
            tasks = []
    else:
        # Попытка миграции из старого формата
        old_tasks = migrate_from_old_format()
        if old_tasks:
            tasks = old_tasks
            save_tasks(tasks)
            print(colorize(f"Миграция выполнена успешно. Импортировано {len(tasks)} задач.", 'green'))
        else:
            log_action("Создан новый пустой список задач")
    return tasks

def save_tasks(tasks: List[Dict[str, Any]]) -> None:
    """
    Сохраняет задачи в JSON-файл. Перед сохранением создаёт резервную копию.
    """
    try:
        # Создаём резервную копию перед сохранением (если есть задачи)
        if tasks:
            create_backup(tasks)
        with open(FILENAME, 'w', encoding='utf-8') as f:
            json.dump(tasks, f, ensure_ascii=False, indent=2)
        log_action(f"Сохранено {len(tasks)} задач")
    except Exception as e:
        log_action(f"Ошибка сохранения: {e}")
        print(colorize(f"Ошибка сохранения: {e}", 'red'))

def print_header(text: str, color: str = 'cyan') -> None:
    """Выводит красивый заголовок с отступами."""
    print("\n" + colorize("=" * 50, color))
    print(colorize(f" {text} ".center(50), color))
    print(colorize("=" * 50, color))

def get_priority_symbol(priority: str) -> str:
    """Возвращает символ для приоритета."""
    symbols = {
        'high': '🔴 ВЫСОКИЙ',
        'medium': '🟡 СРЕДНИЙ',
        'low': '🟢 НИЗКИЙ'
    }
    return symbols.get(priority, '🟡 СРЕДНИЙ')

def format_date(date_str: Optional[str]) -> str:
    """Форматирует дату из ISO в читаемый вид."""
    if not date_str:
        return "—"
    try:
        dt = datetime.fromisoformat(date_str)
        return dt.strftime("%d.%m.%Y")
    except:
        return date_str

def format_task_line(index: int, task: Dict[str, Any], detailed: bool = False) -> str:
    """Форматирует одну задачу для вывода."""
    status = "✓" if task['done'] else "✗"
    status_colored = colorize(status, 'green' if task['done'] else 'red')
    title = task['title']
    priority = get_priority_symbol(task.get('priority', 'medium'))
    if detailed:
        created = format_date(task.get('created_at'))
        due = format_date(task.get('due_date'))
        return f"{index}. [{status_colored}] {title} | Приор: {priority} | Создана: {created} | Дедлайн: {due}"
    else:
        return f"{index}. [{status_colored}] {title} | {priority}"

def view_tasks(tasks: List[Dict[str, Any]], detailed: bool = False) -> None:
    """Просмотр всех задач с возможностью детального вывода."""
    if not tasks:
        print_header("СПИСОК ЗАДАЧ")
        print(colorize("Список задач пуст.", 'yellow'))
        return

    print_header("СПИСОК ЗАДАЧ" + (" (ПОДРОБНО)" if detailed else ""), 'cyan')
    for i, task in enumerate(tasks, 1):
        print(format_task_line(i, task, detailed))

def add_task(tasks: List[Dict[str, Any]]) -> None:
    """Добавление новой задачи с приоритетом и дедлайном."""
    print_header("ДОБАВЛЕНИЕ НОВОЙ ЗАДАЧИ", 'green')
    while True:
        title = input("Введите название задачи (или '0' для отмены): ").strip()
        if title == '0':
            print(colorize("Отмена.", 'yellow'))
            return
        if not title:
            print(colorize("Название не может быть пустым!", 'red'))
            continue
        break

    # Выбор приоритета
    print("\nПриоритет задачи:")
    print("1. Высокий 🔴")
    print("2. Средний 🟡")
    print("3. Низкий 🟢")
    priority_choice = input("Выберите (1-3, по умолчанию 2): ").strip()
    if priority_choice == '1':
        priority = 'high'
    elif priority_choice == '3':
        priority = 'low'
    else:
        priority = 'medium'

    # Ввод дедлайна
    due_date_str = input("Введите дату дедлайна (ДД.ММ.ГГГГ) или оставьте пустым: ").strip()
    due_date = None
    if due_date_str:
        try:
            day, month, year = map(int, due_date_str.split('.'))
            due_date_obj = date(year, month, day)
            due_date = due_date_obj.isoformat()
        except:
            print(colorize("Неверный формат даты. Дедлайн не установлен.", 'yellow'))

    task = {
        'title': title,
        'done': False,
        'priority': priority,
        'created_at': datetime.now().isoformat(),
        'due_date': due_date
    }
    tasks.append(task)
    print(colorize("✅ Задача успешно добавлена!", 'green'))
    log_action(f"Добавлена задача: {title}")

def edit_task(tasks: List[Dict[str, Any]]) -> None:
    """Редактирование задачи: название, приоритет, дедлайн, статус."""
    view_tasks(tasks, detailed=True)
    if not tasks:
        return
    while True:
        choice = input("Введите номер задачи для редактирования (или '0' для отмены): ").strip()
        if choice == '0':
            print(colorize("Отмена.", 'yellow'))
            return
        if choice.isdigit():
            num = int(choice)
            if 1 <= num <= len(tasks):
                task = tasks[num - 1]
                print_header(f"РЕДАКТИРОВАНИЕ: {task['title']}", 'magenta')
                # Редактирование названия
                new_title = input(f"Новое название (пустое - оставить '{task['title']}'): ").strip()
                if new_title:
                    task['title'] = new_title
                # Редактирование приоритета
                print("Новый приоритет (1-высокий, 2-средний, 3-низкий, 0-без изменений):")
                prio = input(f"Текущий: {get_priority_symbol(task['priority'])}: ").strip()
                if prio == '1':
                    task['priority'] = 'high'
                elif prio == '2':
                    task['priority'] = 'medium'
                elif prio == '3':
                    task['priority'] = 'low'
                # Редактирование дедлайна
                current_due = format_date(task.get('due_date'))
                new_due = input(f"Новый дедлайн (ДД.ММ.ГГГГ) или пусто - удалить (текущий: {current_due}): ").strip()
                if new_due == "":
                    task['due_date'] = None
                elif new_due:
                    try:
                        day, month, year = map(int, new_due.split('.'))
                        due_date_obj = date(year, month, day)
                        task['due_date'] = due_date_obj.isoformat()
                    except:
                        print(colorize("Неверный формат, дедлайн не изменён.", 'red'))
                # Редактирование статуса выполнения
                done_choice = input("Отметить как выполненную? (да/нет, текущий: 'Да' если выполнена): ").strip().lower()
                if done_choice == 'да':
                    task['done'] = True
                elif done_choice == 'нет':
                    task['done'] = False
                print(colorize("Задача изменена!", 'green'))
                log_action(f"Отредактирована задача #{num}")
                break
            else:
                print(colorize("Неверный номер задачи.", 'red'))
        else:
            print(colorize("Пожалуйста, введите число.", 'red'))

def delete_task(tasks: List[Dict[str, Any]]) -> None:
    """Удаление задачи по номеру."""
    view_tasks(tasks)
    if not tasks:
        return
    while True:
        choice = input("Введите номер задачи для удаления (или '0' для отмены): ").strip()
        if choice == '0':
            print(colorize("Отмена.", 'yellow'))
            return
        if choice.isdigit():
            num = int(choice)
            if 1 <= num <= len(tasks):
                removed = tasks.pop(num - 1)
                print(colorize(f"Задача '{removed['title']}' удалена!", 'green'))
                log_action(f"Удалена задача: {removed['title']}")
                break
            else:
                print(colorize("Неверный номер задачи.", 'red'))
        else:
            print(colorize("Пожалуйста, введите число.", 'red'))

def mark_done(tasks: List[Dict[str, Any]]) -> None:
    """Отметка задачи как выполненной."""
    view_tasks(tasks)
    if not tasks:
        return
    while True:
        choice = input("Введите номер задачи для отметки как выполненной (или '0' для отмены): ").strip()
        if choice == '0':
            print(colorize("Отмена.", 'yellow'))
            return
        if choice.isdigit():
            num = int(choice)
            if 1 <= num <= len(tasks):
                if not tasks[num - 1]['done']:
                    tasks[num - 1]['done'] = True
                    print(colorize("✅ Задача отмечена как выполненная!", 'green'))
                    log_action(f"Отмечена выполненной: {tasks[num-1]['title']}")
                else:
                    print(colorize("Задача уже выполнена.", 'yellow'))
                break
            else:
                print(colorize("Неверный номер задачи.", 'red'))
        else:
            print(colorize("Пожалуйста, введите число.", 'red'))

def search_tasks(tasks: List[Dict[str, Any]]) -> None:
    """Поиск задач по подстроке в названии."""
    if not tasks:
        print_header("ПОИСК")
        print(colorize("Список задач пуст.", 'yellow'))
        return
    query = input("Введите слово для поиска в задачах: ").strip().lower()
    if not query:
        print(colorize("Ничего не введено.", 'red'))
        return
    found = []
    for i, task in enumerate(tasks, 1):
        if query in task['title'].lower():
            found.append((i, task))
    print_header(f"РЕЗУЛЬТАТЫ ПОИСКА ПО '{query}'", 'cyan')
    if not found:
        print(colorize("Совпадений не найдено.", 'yellow'))
    else:
        for num, task in found:
            print(format_task_line(num, task))

def show_stats(tasks: List[Dict[str, Any]]) -> None:
    """Расширенная статистика: всего, выполнено, по приоритетам, просроченные."""
    total = len(tasks)
    done = sum(1 for t in tasks if t['done'])
    todo = total - done
    percent_done = (done / total * 100) if total > 0 else 0

    # Статистика по приоритетам
    priority_counts = {'high': 0, 'medium': 0, 'low': 0}
    for t in tasks:
        p = t.get('priority', 'medium')
        priority_counts[p] += 1

    # Просроченные задачи (если есть дедлайн и он меньше сегодняшней даты)
    today = date.today()
    overdue = 0
    for t in tasks:
        due = t.get('due_date')
        if due and not t['done']:
            try:
                due_date_obj = datetime.fromisoformat(due).date()
                if due_date_obj < today:
                    overdue += 1
            except:
                pass

    print_header("СТАТИСТИКА ЗАДАЧ", 'blue')
    print(f"📊 Всего задач: {total}")
    print(f"✅ Выполнено: {done} ({percent_done:.1f}%)")
    print(f"⏳ Осталось: {todo}")
    print(f"🔴 Высокий приоритет: {priority_counts['high']}")
    print(f"🟡 Средний приоритет: {priority_counts['medium']}")
    print(f"🟢 Низкий приоритет: {priority_counts['low']}")
    print(f"⚠️ Просроченные задачи: {overdue}")

def filter_tasks(tasks: List[Dict[str, Any]]) -> None:
    """Фильтрация задач по статусу, приоритету или дедлайну."""
    if not tasks:
        print_header("ФИЛЬТРАЦИЯ")
        print(colorize("Список задач пуст.", 'yellow'))
        return
    print_header("ФИЛЬТРАЦИЯ ЗАДАЧ", 'magenta')
    print("Фильтровать по:")
    print("1. Только невыполненные")
    print("2. Только выполненные")
    print("3. По приоритету (высокий/средний/низкий)")
    print("4. С дедлайном (показать только с дедлайном)")
    print("5. Просроченные")
    print("0. Отмена")
    choice = input("Ваш выбор: ").strip()
    filtered = []
    if choice == '1':
        filtered = [t for t in tasks if not t['done']]
        print_header("НЕВЫПОЛНЕННЫЕ ЗАДАЧИ")
    elif choice == '2':
        filtered = [t for t in tasks if t['done']]
        print_header("ВЫПОЛНЕННЫЕ ЗАДАЧИ")
    elif choice == '3':
        prio = input("Приоритет (high/medium/low): ").strip().lower()
        if prio in ('high', 'medium', 'low'):
            filtered = [t for t in tasks if t.get('priority', 'medium') == prio]
            print_header(f"ЗАДАЧИ С ПРИОРИТЕТОМ {prio.upper()}")
        else:
            print(colorize("Неверный приоритет.", 'red'))
            return
    elif choice == '4':
        filtered = [t for t in tasks if t.get('due_date') is not None]
        print_header("ЗАДАЧИ С ДЕДЛАЙНОМ")
    elif choice == '5':
        today = date.today()
        for t in tasks:
            due = t.get('due_date')
            if due and not t['done']:
                try:
                    due_date_obj = datetime.fromisoformat(due).date()
                    if due_date_obj < today:
                        filtered.append(t)
                except:
                    pass
        print_header("ПРОСРОЧЕННЫЕ ЗАДАЧИ")
    elif choice == '0':
        return
    else:
        print(colorize("Неверный выбор.", 'red'))
        return
    if not filtered:
        print(colorize("Нет задач, соответствующих фильтру.", 'yellow'))
    else:
        for i, task in enumerate(filtered, 1):
            print(format_task_line(i, task, detailed=False))

def sort_tasks(tasks: List[Dict[str, Any]]) -> None:
    """Сортировка задач (по дате создания, дедлайну, приоритету, статусу)."""
    if not tasks:
        print_header("СОРТИРОВКА")
        print(colorize("Список задач пуст.", 'yellow'))
        return
    print_header("СОРТИРОВКА ЗАДАЧ", 'cyan')
    print("Сортировать по:")
    print("1. Дате создания (новые сначала)")
    print("2. Дате дедлайна (ближайшие сначала)")
    print("3. Приоритету (высокий → низкий)")
    print("4. Статусу (невыполненные → выполненные)")
    print("0. Отмена")
    choice = input("Ваш выбор: ").strip()
    if choice == '1':
        tasks.sort(key=lambda x: x.get('created_at', ''), reverse=True)
        print(colorize("Отсортировано по дате создания (новые сверху)", 'green'))
    elif choice == '2':
        def due_key(task):
            due = task.get('due_date')
            if due is None:
                return '9999-12-31'  # задачи без дедлайна в конец
            return due
        tasks.sort(key=due_key)
        print(colorize("Отсортировано по дедлайну (ближайшие сверху)", 'green'))
    elif choice == '3':
        priority_order = {'high': 0, 'medium': 1, 'low': 2}
        tasks.sort(key=lambda x: priority_order.get(x.get('priority', 'medium'), 1))
        print(colorize("Отсортировано по приоритету (высокий → низкий)", 'green'))
    elif choice == '4':
        tasks.sort(key=lambda x: x['done'])  # False идут раньше
        print(colorize("Отсортировано по статусу (невыполненные сверху)", 'green'))
    elif choice == '0':
        return
    else:
        print(colorize("Неверный выбор.", 'red'))
        return
    # Показать отсортированный список
    view_tasks(tasks)

def export_to_csv(tasks: List[Dict[str, Any]]) -> None:
    """Экспорт задач в CSV файл."""
    if not tasks:
        print_header("ЭКСПОРТ")
        print(colorize("Нет задач для экспорта.", 'yellow'))
        return
    filename = f"tasks_export_{datetime.now().strftime('%Y%m%d_%H%M%S')}.csv"
    try:
        with open(filename, 'w', newline='', encoding='utf-8') as csvfile:
            fieldnames = ['title', 'done', 'priority', 'created_at', 'due_date']
            writer = csv.DictWriter(csvfile, fieldnames=fieldnames)
            writer.writeheader()
            for task in tasks:
                writer.writerow(task)
        print(colorize(f"Экспорт выполнен в файл {filename}", 'green'))
        log_action(f"Экспорт в CSV: {filename}")
    except Exception as e:
        print(colorize(f"Ошибка экспорта: {e}", 'red'))

def import_from_csv(tasks: List[Dict[str, Any]]) -> None:
    """Импорт задач из CSV файла."""
    filename = input("Введите имя CSV файла для импорта: ").strip()
    if not os.path.exists(filename):
        print(colorize("Файл не найден.", 'red'))
        return
    try:
        with open(filename, 'r', encoding='utf-8') as csvfile:
            reader = csv.DictReader(csvfile)
            imported = 0
            for row in reader:
                # Преобразуем строковые 'True'/'False' в bool
                done = row.get('done', 'False').lower() == 'true'
                task = {
                    'title': row.get('title', 'Без названия'),
                    'done': done,
                    'priority': row.get('priority', 'medium'),
                    'created_at': row.get('created_at', datetime.now().isoformat()),
                    'due_date': row.get('due_date') or None
                }
                tasks.append(task)
                imported += 1
        print(colorize(f"Импортировано {imported} задач из {filename}", 'green'))
        log_action(f"Импорт из CSV: {filename}, задач: {imported}")
    except Exception as e:
        print(colorize(f"Ошибка импорта: {e}", 'red'))

def clear_all(tasks: List[Dict[str, Any]]) -> None:
    """Очистка всего списка задач с подтверждением."""
    if not tasks:
        print_header("ОЧИСТКА СПИСКА")
        print(colorize("Список задач уже пуст.", 'yellow'))
        return
    print_header("ОЧИСТКА СПИСКА", 'red')
    confirm = input(f"Вы уверены, что хотите удалить все {len(tasks)} задач? (да/нет): ").strip().lower()
    if confirm == 'да':
        tasks.clear()
        save_tasks(tasks)
        print(colorize("Все задачи удалены!", 'green'))
        log_action("Очищен весь список задач")
    else:
        print(colorize("Операция отменена.", 'yellow'))

def show_reminders(tasks: List[Dict[str, Any]]) -> None:
    """Показывает напоминания: задачи с дедлайном сегодня или просроченные."""
    today = date.today()
    upcoming = []
    overdue = []
    for task in tasks:
        if task['done']:
            continue
        due = task.get('due_date')
        if due:
            try:
                due_date_obj = datetime.fromisoformat(due).date()
                if due_date_obj == today:
                    upcoming.append(task)
                elif due_date_obj < today:
                    overdue.append(task)
            except:
                pass
    print_header("НАПОМИНАНИЯ", 'magenta')
    if overdue:
        print(colorize("⚠️ ПРОСРОЧЕННЫЕ ЗАДАЧИ:", 'red'))
        for t in overdue:
            print(f"  • {t['title']} (дедлайн был {format_date(t['due_date'])})")
    if upcoming:
        print(colorize("📅 ЗАДАЧИ НА СЕГОДНЯ:", 'yellow'))
        for t in upcoming:
            print(f"  • {t['title']} (дедлайн сегодня)")
    if not overdue and not upcoming:
        print(colorize("Нет активных напоминаний.", 'green'))

def main_menu() -> None:
    """Главное меню программы."""
    tasks = load_tasks()
    # Проверка напоминаний при запуске
    if tasks:
        show_reminders(tasks)

    while True:
        print("\n" + colorize("=" * 50, 'cyan'))
        print(colorize("|        РАСШИРЕННЫЙ МЕНЕДЖЕР ЗАДАЧ        |", 'bold'))
        print(colorize("=" * 50, 'cyan'))
        print("| 1.  Добавить задачу                          |")
        print("| 2.  Просмотреть задачи                       |")
        print("| 3.  Редактировать задачу                     |")
        print("| 4.  Удалить задачу                           |")
        print("| 5.  Отметить как выполненную                 |")
        print("| 6.  Поиск по задачам                         |")
        print("| 7.  Статистика                               |")
        print("| 8.  Очистить список                          |")
        print("| 9.  Фильтрация задач                         |")
        print("| 10. Сортировка                               |")
        print("| 11. Экспорт в CSV                            |")
        print("| 12. Импорт из CSV                            |")
        print("| 13. Напоминания                              |")
        print("| 14. Выход                                    |")
        print(colorize("=" * 50, 'cyan'))

        choice = input("Ваш выбор: ").strip()

        if choice == '1':
            add_task(tasks)
            save_tasks(tasks)
        elif choice == '2':
            # Подробный просмотр или обычный
            sub = input("Показать подробно? (да/нет): ").strip().lower()
            view_tasks(tasks, detailed=(sub == 'да'))
        elif choice == '3':
            edit_task(tasks)
            save_tasks(tasks)
        elif choice == '4':
            delete_task(tasks)
            save_tasks(tasks)
        elif choice == '5':
            mark_done(tasks)
            save_tasks(tasks)
        elif choice == '6':
            search_tasks(tasks)
        elif choice == '7':
            show_stats(tasks)
        elif choice == '8':
            clear_all(tasks)
            save_tasks(tasks)
        elif choice == '9':
            filter_tasks(tasks)
        elif choice == '10':
            sort_tasks(tasks)
            save_tasks(tasks)
        elif choice == '11':
            export_to_csv(tasks)
        elif choice == '12':
            import_from_csv(tasks)
            save_tasks(tasks)
        elif choice == '13':
            show_reminders(tasks)
        elif choice == '14':
            print_header("ВЫХОД ИЗ ПРОГРАММЫ", 'green')
            print(colorize("До свидания!", 'bold'))
            log_action("Программа завершена")
            break
        else:
            print_header("ОШИБКА", 'red')
            print(colorize("Неверный выбор. Пожалуйста, введите число от 1 до 14.", 'red'))

if __name__ == "__main__":
    # Создаём папку для резервных копий при старте
    ensure_backup_dir()
    log_action("Запуск программы")
    try:
        main_menu()
    except KeyboardInterrupt:
        print("\n" + colorize("Программа прервана пользователем.", 'yellow'))
        log_action("Принудительное завершение")
    except Exception as e:
        print(colorize(f"Непредвиденная ошибка: {e}", 'red'))
        log_action(f"Критическая ошибка: {e}")
```
