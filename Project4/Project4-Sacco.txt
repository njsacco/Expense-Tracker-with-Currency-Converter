from flask import Flask, render_template, request, url_for
import secrets
from flask_pymongo import PyMongo
from pymongo import MongoClient
from flask_wtf import FlaskForm
from wtforms import StringField, DecimalField, SelectField, DateField, validators
from wtforms.validators import DataRequired
import requests

app = Flask(__name__)
app.config['SECRET_KEY'] = secrets.token_hex(16)


# Connect to MongoDB database
client = MongoClient("mongodb+srv://Admin32:ucwmNlrDP2MPDvtu@cluster0.mwxvhyy.mongodb.net/")
db = client['db']

# Define form class
class ExpensesForm(FlaskForm):
    description = StringField('Description', validators=[DataRequired()])
    category = SelectField('Category', choices=[('restaurants', 'Restaurants'), ('groceries', 'Groceries'), ('utilities', 'Utilities'),
                                                ('transportation', 'Transportation'), ('entertainment', 'Entertainment'), ('other', 'Other')])
    cost = DecimalField('Cost', validators=[DataRequired()])
    currency = SelectField('Currency', choices=[('USD', 'US Dollar'), ('EUR', 'Euro'), ('GBP', 'British Pound'), ('CAD', 'Canadian Dollar'), ('AUD', 'Australian Dollar'), ('JPY', 'Japanese Yen')])
    date = DateField('Date', validators=[DataRequired()])

# Define route for home page
@app.route('/')
def index():
    my_expenses_db = db.expenses.find()
    expenses_by_category = [
        ("restaurants", get_total_expenses("restaurants")),
        ("groceries", get_total_expenses("groceries")),
        ("utilities", get_total_expenses("utilities")),
        ("transportation", get_total_expenses("transportation")),
        ("entertainment", get_total_expenses("entertainment")),
        ("other", get_total_expenses("other"))
    ]
    total_expenses = sum(expense['cost'] for expense in my_expenses_db)
    return render_template("index.html", expenses=my_expenses_db, expenses_by_category=expenses_by_category, total_expenses=total_expenses)


# Define route for expenses form
@app.route('/addExpenses', methods=['GET', 'POST'])
def addExpenses():
    form = ExpensesForm()
    if form.validate_on_submit():
        cost = float(form.cost.data)
        currency = form.currency.data
        usd_cost = currency_converter(cost, currency)
        expense = {
            'description': form.description.data,
            'category': form.category.data,
            'cost': usd_cost,
            'currency': 'USD',
            'date': form.date.data.strftime('%Y-%m-%d')
        }
        db.expenses.insert_one(expense)
        return render_template('expenseAdded.html')
    return render_template('addExpenses.html', form=form)


# Define function to calculate total expenses for a given category
def get_total_expenses(category):
    my_expenses = db.expenses.find({"category": category})
    total_cost = 0
    for i in my_expenses:
        total_cost += i["cost"]
    return total_cost


# Define function to convert currency to USD
def currency_converter(cost, currency):
    url = "https://api.apilayer.com/currency_data/live?apikey=0xjIfuQaRtH3z7AV9W331OthD3MgxfuS"
    response = requests.get(url).json()
    exchange_rate = response["quotes"]["USD" + currency]
    return round(cost / exchange_rate, 2)


if __name__ == '__main__':
    app.debug = True
    app.run()
