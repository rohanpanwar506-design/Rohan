                """, tuple(data.values()))
            self.load_students()
            self.clear_form()
            messagebox.showinfo("Success", "Student record added successfully.")
        except sqlite3.IntegrityError:
            messagebox.showerror("Duplicate roll number", "This roll number already exists. Use a unique roll number.")

    def update_student(self):
        if self.selected_id is None:
            messagebox.showwarning("Select a record", "Please select a student from the table first.")
            return
        data = self.get_form_data()
        if data is None:
            return
        try:
            with self.connect() as con:
                con.execute("""
                    UPDATE students
                    SET roll_no=?, name=?, course=?, semester=?, phone=?, email=?
                    WHERE id=?
                """, (*data.values(), self.selected_id))
            self.load_students()
            self.clear_form()
            messagebox.showinfo("Success", "Student record updated successfully.")
        except sqlite3.IntegrityError:
            messagebox.showerror("Duplicate roll number", "Another student already uses this roll number.")

    def delete_student(self):
        if self.selected_id is None:
            messagebox.showwarning("Select a record", "Please select a student from the table first.")
            return
        if not messagebox.askyesno("Confirm delete", "Are you sure you want to delete this student record?"):
            return
        with self.connect() as con:
            con.execute("DELETE FROM students WHERE id=?", (self.selected_id,))
        self.load_students()
        self.clear_form()
        self.status_var.set("Student record deleted.")

    def load_students(self):
        for item in self.tree.get_children():
            self.tree.delete(item)
        search = self.search_var.get().strip()
        with self.connect() as con:
            if search:
                rows = con.execute("""
                    SELECT id, roll_no, name, course, semester, phone, email
                    FROM students
                    WHERE roll_no LIKE ? OR name LIKE ? OR course LIKE ?
                    ORDER BY name
                """, (f"%{search}%", f"%{search}%", f"%{search}%")).fetchall()
            else:
                rows = con.execute("""
                    SELECT id, roll_no, name, course, semester, phone, email
                    FROM students ORDER BY name
                """).fetchall()
        for row in rows:
            self.tree.insert("", "end", values=row)
        self.status_var.set(f"Total records shown: {len(rows)}")

    def on_select(self, _event=None):
        selection = self.tree.selection()
        if not selection:
            return
        values = self.tree.item(selection[0], "values")
        self.selected_id = int(values[0])
        for key, value in zip(self.vars, values[1:]):
            self.vars[key].set(value)

    def clear_form(self):
        self.selected_id = None
        for var in self.vars.values():
            var.set("")
        for item in self.tree.selection():
            self.tree.selection_remove(item)

    def show_all(self):
        self.search_var.set("")
        self.load_students()

    def export_csv(self):
        path = filedialog.asksaveasfilename(
            title="Export student records",
            defaultextension=".csv",
            filetypes=[("CSV files", "*.csv")]
        )
        if not path:
            return
        with self.connect() as con:
            rows = con.execute("""
                SELECT roll_no, name, course, semester, phone, email
                FROM students ORDER BY name
            """).fetchall()
        try:
            with open(path, "w", newline="", encoding="utf-8-sig") as file:
                writer = csv.writer(file)
                writer.writerow(["Roll Number", "Student Name", "Course", "Semester", "Phone", "Email"])
                writer.writerows(rows)
            messagebox.showinfo("Export complete", f"Records exported successfully:\n{path}")
        except OSError as error:
            messagebox.showerror("Export failed", str(error))


if __name__ == "__main__":
    root = tk.Tk()
    app = StudentRecordApp(root)
    root.mainloop()
