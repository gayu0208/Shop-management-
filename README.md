# ─── Modules To Import ────────────────────────────────────────────────────────
import mysql.connector as sql
from pwinput import pwinput
import bcrypt
import os
from InquirerPy import inquirer, get_style
import datetime
import time
import csv

from rich.progress import track
from rich.panel import Panel
from rich.text import Text
from rich.console import Console
from rich.theme import Theme
from rich.prompt import IntPrompt, FloatPrompt, Prompt
from rich.columns import Columns
from rich import box
from rich.table import Table
from rich.spinner import Spinner
from rich.live import Live

# ──────────────────────────────────────────────────────────────────────────────

style = get_style(
    {
        "questionmark": "bold red",
        "question": "bold red",
        "answered_question": "red",
        "answer": "green",
        "input": "yellow",
        "options": "yellow",
        "pointer": "blue",
        "amark": "red",
    },
    style_override=False,
)

custom_theme = Theme(
    {
        "success": "bold #76B947",
        "error": "bold #FF004D",
        "border": "#DA0037",
        "heading": "bold #EDEDED",
        "menu": "purple",
        "input": "yellow",
        "background": "#171717",
        "pass": "#DA0037 blink",
        "table": "cyan",
    },
    inherit=False,
)

console = Console(theme=custom_theme)

# ─── Connecting To Mysql Database ──────────────────────────────────────────────────────────────────────────────────────────

try:
    conobj = sql.connect(
        host=os.environ.get("DB_HOST"),
        user=os.environ.get("DB_USER"),
        passwd=os.environ.get("DB_PASSWD"),
        database=os.environ.get("DB_DATABASE"),
    )
    cursor = conobj.cursor()
except Exception as e:
    console.print(f"Could not connect to MySQL database \n{e}", style="error")
    spinner = Spinner(
        "point", text=Text("Terminating the Program", style="error"), style="error"
    )
    with Live(spinner, refresh_per_second=20) as live:
        for i in range(7):
            time.sleep(0.2)
    exit()

# ───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

# ─── Declaring Global Variables ────────────────────────────────────────────────────────────────────────────────────────────

email = ""

cart = []
itemnos_in_cart = []
items_table = []
cart_table = []

items = []
items_no = []

# ───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────


def sign_in(user: str = "customer" or "employee") -> bool:
    global email
    while True:
        email = console.input("[input]Enter your email : ")
        q = f"SELECT {user}_name, passwd FROM {user}_table WHERE email = %s"
        v = (email,)
        cursor.execute(q, v)
        rec = cursor.fetchall()
        if len(rec) == 0:
            console.print(f"{email} does not exist in our database", style="error")
            ch = inquirer.confirm(
                message="Do you want to register ?", style=style
            ).execute()
            if ch:
                if user == "customer":
                    customer_register()
                    console.print(
                        "\nYou can sign in with your credentials now\n", style="success"
                    )

                elif user == "employee":
                    employee_register()
                    console.print(
                        "\nYou can sign in with your credentials now\n", style="success"
                    )

        else:
            hashed = (rec[0][1]).encode("utf-8")
            c = 3
            while c > 0:
                console.print("Enter your password : ", end="", style="pass")
                password = pwinput(prompt="").encode("utf-8")
                if bcrypt.checkpw(password, hashed):
                    console.print(
                        f"\nWelcome {rec[0][0].title()} !!! \n", style="success"
                    )

                    spinner = Spinner(
                        "point",
                        text=Text(f"Continuing to {user} screen", style="green"),
                        style="green",
                    )
                    with Live(spinner, refresh_per_second=20) as live:
                        for i in range(5):
                            time.sleep(0.2)

                    return (email, True)
                else:
                    c -= 1
                    console.print(
                        "Password Incorrect \nPlease Try again", style="error"
                    )
                    console.print(f"You have {c} more chances", style="error")
            else:
                console.print("Chances Over", style="error")
                return (email, False)


def customer_register():
    try:
        console.clear()
        console.print(
            Panel.fit(
                Text("CUSTOMER REGISTRATION", style="heading", justify="center"),
                style="border",
            ),
            justify="center",
        )
        print()
        name = console.input("[input]Enter your name : ")

        cursor.execute("SELECT email FROM customer_table")
        emails = cursor.fetchall()

        while True:
            email = console.input("[input]Enter your email id : ")
            flag = 0
            for i in emails:
                if i[0] == email:
                    flag = 1
            if flag == 1:
                console.print("Email already exists", style="error")
            else:
                break

        while True:
            console.print("Enter your password : ", end="", style="pass")
            password = pwinput(prompt="").encode("utf-8")

            console.print("Confirm your password : ", end="", style="pass")
            con_password = pwinput(prompt="").encode("utf-8")

            if password == con_password:
                console.print("Passwords Match", style="success")
                break
            else:
                console.print("Password are not matching \n", style="error")

        hashed = bcrypt.hashpw(password, bcrypt.gensalt())
        q = "INSERT INTO customer_table (customer_name, email, passwd, items) VALUES (%s, %s, %s, '[]')"
        v = (name, email, hashed)
        cursor.execute(q, v)
        conobj.commit()
        console.print("Registered Successfully", style="success")
        console.input("[yellow]Enter a key to continue")

    except Exception as e:
        console.print(
            f"Failed to Register \nPlease try again later \n{e}", style="error"
        )
        console.input("[yellow]Enter a key to continue")


def employee_register():
    try:
        console.clear()
        console.print(
            Panel.fit(
                Text("EMPLOYEE REGISTRATION", style="heading", justify="center"),
                style="border",
            ),
            justify="center",
        )
        print()
        name = console.input("[input]Enter your name : ")

        cursor.execute("SELECT email FROM employee_table")
        emails = cursor.fetchall()

        while True:
            email = console.input("[input]Enter your email id : ")
            flag = 0
            for i in emails:
                if i[0] == email:
                    flag = 1
            if flag == 1:
                console.print("Email already exists", style="error")
            else:
                break

        while True:
            console.print("Enter your password : ", end="", style="pass")
            password = pwinput(prompt="").encode("utf-8")

            console.print("Confirm your password : ", end="", style="pass")
            con_password = pwinput(prompt="").encode("utf-8")

            if password == con_password:
                console.print("Passwords Match", style="success")
                break
            else:
                console.print("Password are not matching \n", style="error")

        hashed = bcrypt.hashpw(password, bcrypt.gensalt())
        des = console.input("[input]Enter your designation : ")
        sal = FloatPrompt.ask("[yellow]Enter your salary")

        q = "INSERT INTO employee_table (employee_name, email, designation, salary, passwd) VALUES (%s, %s, %s, %s, %s)"
        v = (name, email, des, sal, hashed)
        cursor.execute(q, v)
        conobj.commit()
        console.print("Registered Successfully", style="success")
        console.input("[yellow]Enter a key to continue")

    except Exception as e:
        console.print(
            f"Failed to Register \nPlease try again later \n{e}", style="error"
        )
        console.input("[yellow]Enter a key to continue")


def cart_table_create(cart):
    global cart_table
    cart_table = Table(header_style="bold red", box=box.SIMPLE_HEAD, title="Your Cart")
    cart_table.add_column("Item No", justify="center")
    cart_table.add_column("Item Name", justify="left")
    cart_table.add_column("Category", justify="left")
    cart_table.add_column("Price", justify="right")
    cart_table.add_column("Quantity", justify="right")

    for item in cart:
        cart_table.add_row(str(item[0]), item[1], item[2], str(item[3]), str(item[4]))

    return cart_table


def full_table_create(items):
    full_table = Table(
        header_style="bold red", box=box.SIMPLE_HEAD, title="Items Table"
    )
    full_table.add_column("Item No", justify="center")
    full_table.add_column("Item Name", justify="left")
    full_table.add_column("Category", justify="left")
    full_table.add_column("Price", justify="right")
    full_table.add_column("Stock", justify="right")

    for i in range(len(items)):
        full_table.add_row(
            str(items[i][0]),
            items[i][1],
            items[i][2],
            str(items[i][3]),
            str(items[i][4]),
        )

    return full_table


def items_table_create(items):

    items_table = Table(
        header_style="bold red", box=box.SIMPLE_HEAD, title="Items Table"
    )
    items_table.add_column("Item No", justify="center")
    items_table.add_column("Item Name", justify="left")
    items_table.add_column("Category", justify="left")
    items_table.add_column("Price", justify="right")

    for i in range(len(items)):
        items_table.add_row(
            str(items[i][0]), items[i][1], items[i][2], str(items[i][3])
        )
        items[i] = list(items[i])

    return items_table


def bill_table_create(cart):
    table = Table(header_style="bold red", box=box.SIMPLE_HEAD)
    table.add_column("S No", justify="center")
    table.add_column("Item Name", justify="left")
    table.add_column("Quantity", justify="center")
    table.add_column("Price", justify="right")
    table.add_column("Total", justify="right")

    for i in range(len(cart)):
        table.add_row(
            str(i + 1),
            cart[i][1],
            str(cart[i][4]),
            str(cart[i][3]),
            str(cart[i][3] * cart[i][4]),
        )

    return table


def bill(cart, dt, sub_total, tax, total_cost):
    try:
        console.clear()
        console.print(f"{'Your Bill': ^60}", style="BOLD RED")
        q = "SELECT customer_name FROM customer_table WHERE email = '{}'".format(email)
        cursor.execute(q)
        name = cursor.fetchall()[0][0]

        console.print(f"Customer Name : [blue]{name}", style="success")
        console.print(f"Email Address : [blue]{email}", style="success")
        console.print(
            f"Date and Time of Purchase : [blue]{dt.day}/{dt.month}/{dt.year} - {dt.hour}:{dt.minute}",
            style="success",
        )
        console.print(bill_table_create(cart))
        console.print(f"SubTotal : [blue]{sub_total} Rs", style="success")
        console.print("Shipping : [blue]100 Rs", style="success")
        console.print(f"Taxes :  [blue]{tax} Rs", style="success")
        console.print(f"Total : [blue] {total_cost} \n", style="success")

        ch = inquirer.confirm(
            message="Do you want to save the bill as a csv file ?", style=style
        ).execute()
        if ch:
            bill_csv(name, email, dt, cart, sub_total, tax, total_cost)

        console.print(
            "Thank you for shopping at our online supermarket!", style="success"
        )

    except Exception as e:
        console.print(
            f"Your Bill could not be generated \nPlease try again later \n{e}",
            style="error",
        )


def bill_csv(name, email, dt, cart, sub_total, tax, total_cost):
    try:
        file_name = console.input("[yellow]Enter File name : ") + ".csv"
        file_object = open(file_name, "w", newline="")
        wobj = csv.writer(file_object)

        wobj.writerows(
            [
                ["", "", "BILL"],
                ["Name", name],
                ["Email", email],
                ["Date", f"{dt.day}/{dt.month}/{dt.year} - {dt.hour}:{dt.minute}"],
                [
                    "",
                ],
                ["S No", "Item Name", "Quantity", "Price", "Total"],
            ]
        )

        for i in range(len(cart)):
            wobj.writerow(
                [i + 1, cart[i][1], cart[i][4], cart[i][3], cart[i][3] * cart[i][4]]
            )

        wobj.writerows(
            [
                [
                    "",
                ],
                ["", "", "", "Sub Total", sub_total],
                ["", "", "", "Shipping", 100],
                ["", "", "", "Taxes", tax],
                ["", "", "", "Total", total_cost],
            ]
        )

        console.print(
            f"CSV FILE {file_name} has been generated \nPlease check in the program folder \n",
            style="success",
        )
        file_object.close()

        ch = inquirer.confirm(
            message="Do you want to open the csv file on your system now ?", style=style
        ).execute()
        if ch:
            try:
                os.startfile(file_name)
                console.print("File opened", style="success")
            except Exception as e:
                console.print(
                    f"Could not open the file \nPlease make sure a default app is set for opening csv files on your system \n{e}",
                    style="error",
                )
    except Exception as e:
        console.print(
            f"CSV File was not generated \nPlease Try again \n{e}", style="error"
        )
        console.input("[yellow]Enter a key to continue ")


def search_buy():
    global items, itemnos_in_cart
    q = "SELECT item_no, item_name, category, price, stock FROM items_table WHERE stock > 0"
    cursor.execute(q)
    items = cursor.fetchall()
    item_names = []
    for item in items:
        item_names.append(item[1])

    try:
        while True:
            console.clear()
            console.print(
                Panel.fit(
                    Text("SEARCH AND BUY", style="heading", justify="center"),
                    style="border",
                ),
                justify="center",
            )
            search = inquirer.fuzzy(
                message="Search for products",
                choices=item_names,
                style=style,
                border=True,
            ).execute()
            while True:
                flag = False
                qty = IntPrompt.ask("[yellow]Enter quantity")
                if qty == 0:
                    console.print(
                        "Quantity cannot be zero \nPlease Enter a non-zero number",
                        style="error",
                    )
                else:
                    for item in items:
                        if item[1] == search:
                            if item[4] < qty:
                                console.print(
                                    f"Limited Stock left \nPlease enter a value less than {item[4] + 1}",
                                    style="error",
                                )
                                flag = True
                                break
                    if not flag:
                        break

            for item in items:
                if item[1] == search:
                    console.print(
                        f"\n[success][i]ITEM DETAILS : [/i][/success]\n Item no : {item[0]} \n Item name : {item[1]} \n Category : {item[2]} \n Price : {item[3]}",
                        style="blue",
                    )
                    brought = list(item)[0:-1]
                    brought.append(qty)
                    break

            itemnos_in_cart.append(str(brought[0]))
            cart.append(brought)
            item_names.remove(search)

            console.print("\nItem added to your cart", style="success")
            ch = inquirer.confirm(message="Continue Shopping ? ", style=style).execute()
            if not ch:
                console.print(
                    "All items successfully added to your cart", style="success"
                )
                spinner = Spinner(
                    "point",
                    text=Text(f"Continuing to confirm purchase", style="green"),
                    style="green",
                )
                with Live(spinner, refresh_per_second=20) as live:
                    for i in range(7):
                        time.sleep(0.2)
                break
    except Exception as e:
        console.print(f"Could not perform the action \n{e}", style="error")
        console.input("[yellow]Enter a key to continue ")


def confirm_purchase():
    global items, cart, itemnos_in_cart

    console.clear()

    console.print(
        Panel.fit(
            Text("CONFIRMATION MENU", style="heading", justify="center"), style="border"
        ),
        justify="center",
    )

    console.print(cart_table_create(cart))

    sub_total = 0
    for i in cart:
        sub_total += i[3] * i[4]
    tax = sub_total / 10
    total_cost = sub_total + tax + 100

    console.print(f"Total Bill = [blue]{sub_total}", style="success")
    ch = inquirer.confirm(message="Confirm your purchase ", style=style).execute()
    if ch:
        try:
            q = "SELECT * FROM customer_table WHERE email = %s"
            v = (email,)
            cursor.execute(q, v)
            userinfo = cursor.fetchall()

            db_purchase = userinfo[0][4]
            db_purchase += total_cost

            db_points = userinfo[0][5]
            db_points += (total_cost / 100) * 5

            current_order = []
            dt = datetime.datetime.now()
            current_order.append(
                f"{dt.day}/{dt.month}/{dt.year} - {dt.hour}:{dt.minute}"
            )
            for i in itemnos_in_cart:
                current_order.append(int(i))

            items = eval(userinfo[0][3])
            items.append(current_order)

            for item in cart:
                q = "UPDATE items_table SET stock = stock - %s where item_no = %s"
                v = (item[4], item[0])
                cursor.execute(q, v)
                conobj.commit()

            q = "UPDATE customer_table SET total_purchase = %s, points = %s, items = %s WHERE email = %s"
            v = (db_purchase, db_points, str(items), email)
            cursor.execute(q, v)
            conobj.commit()
            console.print("Purchase Confirmed \n", style="success")
            console.print("Generating your bill")
            bill(cart, dt, sub_total, tax, total_cost)
            console.input("[success]Enter a key to continue")

        except Exception as e:
            console.print(
                f"Failed to Confirm your Purchase \nPlease try again later \n{e}",
                style="error",
            )
            console.input("[success]Enter a key to continue")

    else:
        console.print("Purchase cancelled", style="error")
        console.input("[success]Enter a key to continue")

    cart = []
    itemnos_in_cart = []


def edit_quantity(items, cart):
    console.clear()

    console.print(
        Panel.fit(
            T
