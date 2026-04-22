# ne-beyte-pojaluysta
Итак, по итогу было решено сделать простенькую программу по типу блокнота. В данном блакноте можно добавлять задачи, уберать задачи, делать очистку задачи, посмотреть задачу, отредактировать, посмотреть статистику выполненых и удалённых задач.
```shell
FILENAME = 'tasks.txt'

def load_tasks():
    tasks = []
    try:
        file = open(FILENAME, 'r', encoding='utf-8')
        lines = file.readlines()
        for line in lines:
            line = line.strip()
            if line:
                parts = line.split('|')
                if len(parts) == 2:
                    title, status = parts
                    tasks.append({'title': title, 'done': status == 'True'})
        file.close()
    except FileNotFoundError:
        pass
    return tasks

def save_tasks(tasks):
    file = open(FILENAME, 'w', encoding='utf-8')
    for task in tasks:
        file.write(f"{task['title']}|{str(task['done'])}\n")
    file.close()

def print_header(text):
    print("\n" + "=" * 40)
    print(f" {text} ")
    print("=" * 40)

def add_task(tasks):
    print_header("ДОБАВЛЕНИЕ НОВОЙ ЗАДАЧИ")
    while True:
        title = input("Введите название задачи (или '0' для отмены): ").strip()
        if title == '0':
            print("Отмена.")
            return
        if title:
            tasks.append({'title': title, 'done': False})
            print("Задача успешно добавлена!")
            break
        else:
            print("Название не может быть пустым!")

def view_tasks(tasks):
    if not tasks:
        print_header("СПИСОК ЗАДАЧ")
        print("Список задач пуст.")
        return

    print_header("СПИСОК ЗАДАЧ")
    for i, task in enumerate(tasks, 1):
        status = "✓ ВЫПОЛНЕНО" if task['done'] else "✗ НЕ ВЫПОЛНЕНО"
        print(f"{i}. [{status}] {task['title']}")

def mark_done(tasks):
    view_tasks(tasks)
    if not tasks:
        return
    while True:
        choice = input("Введите номер задачи для отметки как выполненная (или '0' для отмены): ").strip()
        if choice == '0':
            print("Отмена.")
            return
        if choice.isdigit():
            num = int(choice)
            if 1 <= num <= len(tasks):
                tasks[num - 1]['done'] = True
                print("Задача отмечена как выполненная!")
                break
            else:
                print("Неверный номер задачи.")
        else:
            print("Пожалуйста, введите число.")

def delete_task(tasks):
    view_tasks(tasks)
    if not tasks:
        return
    while True:
        choice = input("Введите номер задачи для удаления (или '0' для отмены): ").strip()
        if choice == '0':
            print("Отмена.")
            return
        if choice.isdigit():
            num = int(choice)
            if 1 <= num <= len(tasks):
                removed = tasks.pop(num - 1)
                print(f"Задача '{removed['title']}' удалена!")
                break
            else:
                print("Неверный номер задачи.")
        else:
            print("Пожалуйста, введите число.")

def edit_task(tasks):
    view_tasks(tasks)
    if not tasks:
        return
    while True:
        choice = input("Введите номер задачи для редактирования (или '0' для отмены): ").strip()
        if choice == '0':
            print("Отмена.")
            return
        if choice.isdigit():
            num = int(choice)
            if 1 <= num <= len(tasks):
                new_title = input(f"Введите новое название для задачи '{tasks[num-1]['title']}': ").strip()
                if new_title:
                    tasks[num - 1]['title'] = new_title
                    print("Задача изменена!")
                    break
                else:
                    print("Название не может быть пустым!")
            else:
                print("Неверный номер задачи.")
        else:
            print("Пожалуйста, введите число.")

def search_tasks(tasks):
    if not tasks:
        print_header("ПОИСК")
        print("Список задач пуст.")
        return
    query = input("Введите слово для поиска в задачах: ").strip().lower()
    if not query:
        print("Ничего не введено.")
        return
    found = []
    for i, task in enumerate(tasks, 1):
        if query in task['title'].lower():
            found.append((i, task))
    print_header(f"РЕЗУЛЬТАТЫ ПОИСКА ПО '{query}'")
    if not found:
        print("Совпадений не найдено.")
    else:
        for num, task in found:
            status = "✓" if task['done'] else "✗"
            print(f"{num}. [{status}] {task['title']}")

def show_stats(tasks):
    total = len(tasks)
    done = sum(1 for t in tasks if t['done'])
    todo = total - done
    percent_done = (done / total * 100) if total > 0 else 0
    print_header("СТАТИСТИКА ЗАДАЧ")
    print(f"Всего задач: {total}")
    print(f"Выполнено: {done}")
    print(f"Осталось: {todo}")
    print(f"Процент выполнения: {percent_done:.1f}%")

def clear_all(tasks):
    if not tasks:
        print_header("ОЧИСТКА СПИСКА")
        print("Список задач уже пуст.")
        return
    print_header("ОЧИСТКА СПИСКА")
    confirm = input(f"Вы уверены, что хотите удалить все {len(tasks)} задач? (да/нет): ").strip().lower()
    if confirm == 'да':
        tasks.clear()
        save_tasks(tasks)
        print("Все задачи удалены!")
    else:
        print("Операция отменена.")

def main_menu():
    tasks = load_tasks()
    while True:
        print("\n" + "=" * 40)
        print("| ГЛАВНОЕ МЕНЮ МЕНЕДЖЕРА ЗАДАЧ |")
        print("| 1. Добавить задачу          |")
        print("| 2. Просмотреть задачи       |")
        print("| 3. Редактировать задачу     |")
        print("| 4. Удалить задачу           |")
        print("| 5. Отметить как выполненную |")
        print("| 6. Поиск по задачам         |")
        print("| 7. Статистика               |")
        print("| 8. Очистить список          |")
        print("| 9. Выход                    |")
        print("=" * 40)
        
        choice = input("Ваш выбор: ").strip()
        
        if choice == '1':
            add_task(tasks)
            save_tasks(tasks)
        elif choice == '2':
            view_tasks(tasks)
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
            print_header("ВЫХОД ИЗ ПРОГРАММЫ")
            print("До свидания!")
            break
        else:
            print_header("ОШИБКА")
            print("Неверный выбор. Пожалуйста, введите число от 1 до 9.")

if __name__ == "__main__":
```
