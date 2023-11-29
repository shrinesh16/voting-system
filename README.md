import mysql.connector as c
import tkinter as tk
from tkinter import ttk, messagebox, simpledialog

class DatabaseManager:
    def __init__(self, user, host, passwd, database):
        self.con = c.connect(user=user, host=host, passwd=passwd)
        self.cur = self.con.cursor()
        self.cur.execute(f"create database if not exists {database}")
        self.con.commit()
        self.cur.execute(f"use {database}")
        self.cur.execute("create table if not exists vote (party varchar(100), votes int(100))")
        self.cur.execute("create table if not exists voters (id INT AUTO_INCREMENT PRIMARY KEY, name VARCHAR(255), voter_id VARCHAR(255) UNIQUE)")
        self.con.commit()

    def execute_query(self, query, params=None):
        try:
            self.cur.execute(query, params)
            self.con.commit()
            return True
        except Exception as e:
            messagebox.showerror("ERROR!!!", f"We ran into some error --> {str(e)} please try again.")
            return False

    def fetch_data(self, query, params=None):
        self.cur.execute(query, params)
        return self.cur.fetchall()

class VoterRegistration:
    def __init__(self, db_manager):
        self.db_manager = db_manager

    def register_voter(self, name, voter_id):
        query = "INSERT INTO voters (name, voter_id) VALUES (%s, %s)"
        return self.db_manager.execute_query(query, (name, voter_id))

    def add_voter_id(self, name, voter_id):
        return self.register_voter(name, voter_id)

    def remove_voter_id(self, voter_id):
        query = "DELETE FROM voters WHERE voter_id = %s"
        return self.db_manager.execute_query(query, (voter_id,))

class UIHandler:
    def __init__(self, root, db_manager, voter_registration):
        self.root = root
        self.db_manager = db_manager
        self.voter_registration = voter_registration
        self.root.geometry("400x400")
        self.root.configure(bg="black")
        self.root.title("Voting System")

        self.passwd = "SRMIST"
        self.create_widgets()

    def create_widgets(self):
        self.label = tk.Label(self.root, text="WELCOME TO SRM VOTING BOOTH", font="bold", fg="white", bg="black")
        self.label.pack()

        self.register_button = tk.Button(self.root, text="Register Voter", width=30, command=self.register_voter)
        self.register_button.pack()

        self.vote_button = tk.Button(self.root, text="VOTE", width=30, command=self.vote)
        self.vote_button.pack()

        self.admin_button = tk.Button(self.root, text="Admin", width=30, command=self.admin)
        self.admin_button.pack()

    def register_voter(self):
        self.root.withdraw()  # Hide the main window

        window_register = tk.Toplevel(self.root)
        window_register.geometry("600x200")
        window_register.title("Voter Registration")

        l1 = tk.Label(window_register, text="Name", width=20, font="bold")
        l1.grid(row=1, column=1)

        e1 = tk.Entry(window_register, width=30)
        e1.grid(row=1, column=2)

        l2 = tk.Label(window_register, text="Voter ID", width=20, font="bold")
        l2.grid(row=2, column=1)

        e2 = tk.Entry(window_register, width=30)
        e2.grid(row=2, column=2)

        b1 = tk.Button(window_register, text="Register", width=20, command=lambda: self.register_true(e1, e2))
        b1.grid(row=3, column=2)

    def register_true(self, e1, e2):
        name = e1.get()
        voter_id = e2.get()

        if name and voter_id:
            if self.voter_registration.register_voter(name, voter_id):
                messagebox.showinfo("Success", "Voter registered successfully.")
            else:
                messagebox.showerror("ERROR!!!", "Voter registration failed.")
        else:
            messagebox.showerror("ERROR!!!", "Please enter both Name and Voter ID.")

    def vote(self):
        voter_id = self.get_voter_id()
        if voter_id:
            self.root.withdraw()  # Hide the main window

            window2 = tk.Toplevel(self.root)
            window2.geometry("900x400")
            window2.title("Vote")

            l1 = tk.Label(window2, text="CHOOSE YOUR VOTE", width=20, font="bold")
            l1.grid(row=1, column=1)

            ls = self.get_party_list()
            e1 = tk.StringVar(window2)
            e1.set("Select party")
            drop = tk.OptionMenu(window2, e1, *ls)
            drop.grid(row=1, column=3)

            b1 = tk.Button(window2, text="VOTE", width=20, command=lambda: self.vote_true(e1, voter_id))
            b1.grid(row=2, column=2)

            # Bind the Enter key to the "Proceed" button
            window2.bind('<Return>', lambda event=None: b1.invoke())

    def get_voter_id(self):
        voter_id = simpledialog.askstring("Voter ID", "Enter your Voter ID:")
        if voter_id:
            query = "SELECT * FROM voters WHERE voter_id = %s"
            result = self.db_manager.fetch_data(query, (voter_id,))
            if result:
                return voter_id
            else:
                messagebox.showerror("ERROR!!!", "Voter ID not found. Please register first.")
                return None

    def get_party_list(self):
        data = self.db_manager.fetch_data("select party from vote")
        return [i[0] for i in data]

    def vote_true(self, e1, voter_id):
        try:
            party = e1.get()
            cmd = "update vote set votes = votes + 1 where party = %s"
            self.db_manager.execute_query(cmd, (party,))
            messagebox.showinfo("SUCCESS", "Successfully voted ")

            # Close the current window
            self.root.destroy()

        except Exception as e:
            messagebox.showerror("ERROR!!!", f"We ran into some error --> {str(e)} please try again.")

    def admin(self):
        self.root.withdraw()  # Hide the main window

        window3 = tk.Toplevel(self.root)
        window3.geometry("700x300")
        window3.title("Admin Login")

        l1 = tk.Label(window3, text="Enter Admin Password", width=30, font="bold")
        l1.grid(row=1, column=1)

        e1 = tk.Entry(window3, width=20, show='*')
        e1.grid(row=1, column=2)

        b1 = tk.Button(window3, text="Proceed", width=30, command=lambda: self.admin_true(window3, e1))
        b1.grid(row=2, column=2)

        # Bind the Enter key to the "Proceed" button
        window3.bind('<Return>', lambda event=None: b1.invoke())

    def admin_true(self, window, e1):
        new_passwd = e1.get()
        if new_passwd == self.passwd:
            messagebox.showinfo("Success", "Login Successful ... Redirecting to Admin page")
            window.destroy()
            self.show_admin_section()
        else:
            messagebox.showerror("ERROR", "Incorrect Password. Please try again")

    def show_admin_section(self):
        window4 = tk.Toplevel(self.root)
        window4.geometry("400x400")
        window4.title("Admin Section")

        b1 = tk.Button(window4, text="Add party", width=30, command=self.add_party)
        b1.pack()

        b2 = tk.Button(window4, text="Remove party", width=30, command=self.remove_party)
        b2.pack()

        b3 = tk.Button(window4, text="Show Result", width=30, command=self.show_result)
        b3.pack()

        b4 = tk.Button(window4, text="Add Voter ID", width=30, command=self.add_voter_id_admin)
        b4.pack()

        b5 = tk.Button(window4, text="Remove Voter ID", width=30, command=self.remove_voter_id_admin)
        b5.pack()

    def add_party(self):
        window5 = tk.Toplevel(self.root)
        window5.geometry("700x300")
        window5.title("Add Party")

        l1 = tk.Label(window5, text="Party Name", width=30, font="bold")
        l1.grid(row=1, column=1)

        e1 = tk.Entry(window5, width=30)
        e1.grid(row=1, column=2)

        b1 = tk.Button(window5, text="Proceed", width=30, command=lambda: self.add_party_true(e1))
        b1.grid(row=2, column=1, columnspan=2)

        # Bind the Enter key to the "Proceed" button
        window5.bind('<Return>', lambda event=None: b1.invoke())

    def add_party_true(self, e1):
        try:
            party = e1.get()
            cmd = "insert into vote values(%s, 0)"
            self.db_manager.execute_query(cmd, (party,))
            messagebox.showinfo("Success", "Party added to the record successfully")

        except Exception as e:
            messagebox.showerror("ERROR!!!", f"We ran into some error --> {str(e)} please try again.")

    def remove_party(self):
        window6 = tk.Toplevel(self.root)
        window6.geometry("700x300")
        window6.title("Remove Party")

        l1 = tk.Label(window6, text="Party Name", width=30, font="bold")
        l1.grid(row=1, column=1)

        e1 = tk.Entry(window6, width=30)
        e1.grid(row=1, column=2)

        b1 = tk.Button(window6, text="Proceed", width=30, command=lambda: self.remove_party_true(e1))
        b1.grid(row=2, column=1, columnspan=2)

        # Bind the Enter key to the "Proceed" button
        window6.bind('<Return>', lambda event=None: b1.invoke())

    def remove_party_true(self, e1):
        try:
            party = e1.get()
            cmd = "delete from vote where party = %s"
            self.db_manager.execute_query(cmd, (party,))
            messagebox.showinfo("Success", "Party removed from the record successfully")

        except Exception as e:
            messagebox.showerror("ERROR!!!", f"We ran into some error --> {str(e)} please try again.")

    def show_result(self):
        window6 = tk.Toplevel(self.root)
        window6.geometry("400x600")
        window6.title("Voting Results")

        tree = ttk.Treeview(window6, columns=("c1", "c2"), show="headings", height=10)
        tree.column("#1", anchor="center")
        tree.heading("#1", text="Party")
        tree.column("#2", anchor="center")
        tree.heading("#2", text="Votes")

        cmd = "select * from vote"
        data = self.db_manager.fetch_data(cmd)
        c = 1
        for i in data:
            tree.insert('', 'end', text=str(c), values=(i[0], i[1]))
            c += 1

        tree.grid(row=1, column=1)

    def add_voter_id_admin(self):
        name = simpledialog.askstring("Voter Registration", "Enter Voter Name:")
        if name:
            voter_id = simpledialog.askstring("Voter Registration", "Enter Voter ID:")
            if voter_id:
                if self.voter_registration.add_voter_id(name, voter_id):
                    messagebox.showinfo("Success", "Voter ID added to the system successfully.")
                else:
                    messagebox.showerror("ERROR!!!", "Failed to add Voter ID.")
            else:
                messagebox.showerror("ERROR!!!", "Please enter Voter ID.")
        else:
            messagebox.showerror("ERROR!!!", "Please enter Voter Name.")

    def remove_voter_id_admin(self):
        voter_id = simpledialog.askstring("Remove Voter ID", "Enter Voter ID:")
        if voter_id:
            if self.voter_registration.remove_voter_id(voter_id):
                messagebox.showinfo("Success", "Voter ID removed from the system successfully.")
            else:
                messagebox.showerror("ERROR!!!", "Failed to remove Voter ID.")
        else:
            messagebox.showerror("ERROR!!!", "Please enter Voter ID.")

if __name__ == "__main__":
    db_manager = DatabaseManager(user='root', host="localhost", passwd="Whiterice", database="voting")
    voter_registration = VoterRegistration(db_manager)
    root = tk.Tk()
    app = UIHandler(root, db_manager, voter_registration)
    root.mainloop()
mmm
