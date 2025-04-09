import tkinter as tk
from tkinter import ttk, messagebox
from tkcalendar import DateEntry
import json
import os
from datetime import datetime
from reportlab.pdfgen import canvas
from reportlab.lib.pagesizes import letter

TASK_FILE = "advanced_tasks.json"

class ToDoApp:
    def __init__(self, root):
        self.root = root
        self.root.title("📜 To-Do List")
        self.root.configure(bg="#F2F4F7")
        self.root.state('zoomed')

        self.tasks = []
        self.dark_mode = False

        self.setup_ui()
        self.load_tasks()

    def setup_ui(self):
        title_frame = tk.Frame(self.root, bg="#4A90E2")
        title_frame.pack(fill='x')
        title_label = tk.Label(title_frame, text="📜 To-Do List", font=("Segoe UI", 24, "bold"), bg="#4A90E2", fg="white", pady=10)
        title_label.pack()

        input_frame = tk.LabelFrame(self.root, text="Task Details", bg="#F2F4F7", font=("Segoe UI", 12, "bold"), fg="#333")
        input_frame.pack(fill='x', padx=20, pady=10)

        tk.Label(input_frame, text="Task:", bg="#F2F4F7", font=("Segoe UI", 10)).grid(row=0, column=0, padx=10, pady=5)
        self.task_entry = tk.Entry(input_frame, width=50, font=("Segoe UI", 10))
        self.task_entry.grid(row=0, column=1, padx=10, pady=5)

        tk.Label(input_frame, text="Due Date:", bg="#F2F4F7", font=("Segoe UI", 10)).grid(row=0, column=2, padx=10, pady=5)
        self.date_entry = DateEntry(input_frame, width=12, font=("Segoe UI", 10), background='darkblue', foreground='white', borderwidth=2, date_pattern='dd-mm-yyyy')
        self.date_entry.set_date(datetime.now())
        self.date_entry.grid(row=0, column=3, padx=10, pady=5)

        tk.Label(input_frame, text="Priority:", bg="#F2F4F7", font=("Segoe UI", 10)).grid(row=0, column=4, padx=10, pady=5)
        self.priority_combo = ttk.Combobox(input_frame, values=["High", "Medium", "Low"], font=("Segoe UI", 10))
        self.priority_combo.grid(row=0, column=5, padx=10, pady=5)

        tk.Label(input_frame, text="Category:", bg="#F2F4F7", font=("Segoe UI", 10)).grid(row=0, column=6, padx=10, pady=5)
        self.category_combo = ttk.Combobox(input_frame, values=["Work", "Personal", "Shopping", "Others"], font=("Segoe UI", 10))
        self.category_combo.grid(row=0, column=7, padx=10, pady=5)

        button_frame = tk.LabelFrame(self.root, text="Actions", bg="#F2F4F7", font=("Segoe UI", 12, "bold"), fg="#333")
        button_frame.pack(fill='x', padx=20, pady=5)

        tk.Button(button_frame, text="Add Task", command=self.add_task, bg="#4CAF50", fg="white", width=12).pack(side='left', padx=5, pady=5)
        tk.Button(button_frame, text="Edit Task", command=self.edit_task, bg="#2196F3", fg="white", width=12).pack(side='left', padx=5, pady=5)
        tk.Button(button_frame, text="Mark Completed", command=self.mark_completed, bg="#FF9800", fg="white", width=15).pack(side='left', padx=5, pady=5)
        tk.Button(button_frame, text="Clear Completed", command=self.clear_completed, bg="#9C27B0", fg="white", width=15).pack(side='left', padx=5, pady=5)
        tk.Button(button_frame, text="Delete Task", command=self.delete_task, bg="#F44336", fg="white", width=12).pack(side='left', padx=5, pady=5)
        tk.Button(button_frame, text="Export to PDF", command=self.export_to_pdf, bg="#607D8B", fg="white", width=15).pack(side='left', padx=5, pady=5)
        tk.Button(button_frame, text="Toggle Dark Mode", command=self.toggle_dark_mode, bg="#795548", fg="white", width=18).pack(side='left', padx=5, pady=5)

        search_frame = tk.LabelFrame(self.root, text="Search & Sort", bg="#F2F4F7", font=("Segoe UI", 12, "bold"), fg="#333")
        search_frame.pack(fill='x', padx=20, pady=10)

        self.search_entry = tk.Entry(search_frame, font=("Segoe UI", 10))
        self.search_entry.pack(side='left', padx=(10, 0), pady=5, ipadx=30)
        tk.Button(search_frame, text="🔍", command=self.search_tasks).pack(side='left', padx=5)

        self.sort_combo = ttk.Combobox(search_frame, values=["Due Date", "Priority"], font=("Segoe UI", 10))
        self.sort_combo.set("Sort By")
        self.sort_combo.pack(side='left', padx=10)
        tk.Button(search_frame, text="Sort", command=self.sort_tasks).pack(side='left', padx=5)

        self.tree = ttk.Treeview(self.root, columns=("Task", "Due Date", "Priority", "Category", "Status"), show="headings")
        for col in ("Task", "Due Date", "Priority", "Category", "Status"):
            self.tree.heading(col, text=col)
            self.tree.column(col, width=150, anchor='center')
        self.tree.pack(fill='both', expand=True, padx=20, pady=10)

        style = ttk.Style()
        style.configure("Treeview", font=("Segoe UI", 10))
        style.configure("Treeview.Heading", font=("Segoe UI", 11, "bold"))

    def add_task(self):
        task_name = self.task_entry.get()
        due_date = self.date_entry.get_date().strftime("%d-%m-%Y")
        priority = self.priority_combo.get()
        category = self.category_combo.get()

        if not task_name:
            messagebox.showwarning("Input Error", "Please enter a task name.")
            return

        for task in self.tasks:
            if (
                task['name'].lower() == task_name.lower() and
                task['due_date'] == due_date and
                task['priority'].lower() == priority.lower() and
                task['category'].lower() == category.lower()
            ):
                messagebox.showwarning(
                    "Duplicate Task",
                    f"Task '{task_name}' with same due date, priority and category already exists."
                )
                return

        self.tasks.append({
            "name": task_name,
            "due_date": due_date,
            "priority": priority,
            "category": category,
            "completed": False
        })
        self.save_tasks()
        self.refresh_tree()
        self.clear_inputs()

    def edit_task(self):
        selected = self.tree.selection()
        if not selected:
            messagebox.showwarning("Select Task", "Please select a task to edit.")
            return

        index = self.tree.index(selected[0])
        self.tasks[index] = {
            "name": self.task_entry.get(),
            "due_date": self.date_entry.get_date().strftime("%d-%m-%Y"),
            "priority": self.priority_combo.get(),
            "category": self.category_combo.get(),
            "completed": self.tasks[index]['completed']
        }
        self.save_tasks()
        self.refresh_tree()
        self.clear_inputs()

    def mark_completed(self):
        selected = self.tree.selection()
        if not selected:
            messagebox.showwarning("Select Task", "Please select a task to mark completed.")
            return

        index = self.tree.index(selected[0])
        self.tasks[index]['completed'] = True
        self.save_tasks()
        self.refresh_tree()

    def clear_completed(self):
        self.tasks = [task for task in self.tasks if not task['completed']]
        self.save_tasks()
        self.refresh_tree()

    def delete_task(self):
        selected = self.tree.selection()
        if not selected:
            messagebox.showwarning("Select Task", "Please select a task to delete.")
            return

        index = self.tree.index(selected[0])
        del self.tasks[index]
        self.save_tasks()
        self.refresh_tree()

    def refresh_tree(self):
        for item in self.tree.get_children():
            self.tree.delete(item)

        for task in self.tasks:
            color = {"High": "red", "Medium": "orange", "Low": "green"}.get(task['priority'], "black")
            status = "Completed" if task['completed'] else "Pending"
            self.tree.insert("", "end", values=(task['name'], task['due_date'], task['priority'], task['category'], status), tags=(color,))

        self.tree.tag_configure("red", foreground="red")
        self.tree.tag_configure("orange", foreground="orange")
        self.tree.tag_configure("green", foreground="green")

    def clear_inputs(self):
        self.task_entry.delete(0, tk.END)
        self.priority_combo.set("")
        self.category_combo.set("")

    def save_tasks(self):
        with open(TASK_FILE, "w") as f:
            json.dump(self.tasks, f)

    def load_tasks(self):
        if os.path.exists(TASK_FILE):
            with open(TASK_FILE, "r") as f:
                self.tasks = json.load(f)
        self.refresh_tree()

    def search_tasks(self):
        query = self.search_entry.get().lower()
        for item in self.tree.get_children():
            self.tree.delete(item)

        for task in self.tasks:
            if query in task['name'].lower():
                color = {"High": "red", "Medium": "orange", "Low": "green"}.get(task['priority'], "black")
                status = "Completed" if task['completed'] else "Pending"
                self.tree.insert("", "end", values=(task['name'], task['due_date'], task['priority'], task['category'], status), tags=(color,))

    def sort_tasks(self):
        sort_by = self.sort_combo.get()
        if sort_by == "Due Date":
            self.tasks.sort(key=lambda x: datetime.strptime(x['due_date'], "%d-%m-%Y"))
        elif sort_by == "Priority":
            priority_order = {"High": 1, "Medium": 2, "Low": 3}
            self.tasks.sort(key=lambda x: priority_order.get(x['priority'], 4))
        self.refresh_tree()

    def export_to_pdf(self):
        c = canvas.Canvas("tasks.pdf", pagesize=letter)
        width, height = letter
        y = height - 50
        c.setFont("Helvetica", 14)
        c.drawString(50, y, "Task List")
        y -= 30
        c.setFont("Helvetica", 10)
        for task in self.tasks:
            line = f"{task['name']} - {task['due_date']} - {task['priority']} - {task['category']} - {'Completed' if task['completed'] else 'Pending'}"
            c.drawString(50, y, line)
            y -= 20
        c.save()
        messagebox.showinfo("Exported", "Tasks exported to tasks.pdf")

    def toggle_dark_mode(self):
        if self.dark_mode:
            self.root.configure(bg="#F2F4F7")
            self.dark_mode = False
        else:
            self.root.configure(bg="#2C2C2C")
            self.dark_mode = True

if __name__ == '__main__':
    root = tk.Tk()
    app = ToDoApp(root)
    root.mainloop()

