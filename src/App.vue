<template>
  <div class="app-container">
    <h1 class="title">Expense Tracker</h1>

    <div class="budget-container">
      <label>Set Daily Budget: </label>
      <input
        type="number"
        v-model.number="budgetLimit"
        class="input-field-budget"
        @change="validateBudget"
      />
    </div>

    <div class="input-container">
      <input type="text" v-model="newExpense.name" placeholder="Expense Name" class="input-field" />
      <input
        type="number"
        v-model.number="newExpense.amount"
        placeholder="Amount"
        class="input-field"
      />
      <div class="date-container">
        <p class="date-label">Date:</p>
        <input type="date" v-model="newExpense.date" class="date-input" />
      </div>
      <button @click="addExpense" class="add-btn">Add Expense</button>
      <p v-if="errorMessage" class="error">{{ errorMessage }}</p>
    </div>

    <div class="summary">
      <h3>Today's Expenses: ₹{{ totalSpentToday.toLocaleString('en-IN') }}</h3>
      <div class="savings">
      <p v-if="remainingBalance > 0" class="balance positive">
        Today's remaining balance: +₹{{ remainingBalance.toLocaleString('en-IN') }}
      </p>
      <p v-if="remainingBalance < 0" class="balance negative">
        Today's deficit: -₹{{ Math.abs(remainingBalance).toLocaleString('en-IN') }}
      </p>

      <h3 v-if="totalSavingsThisMonth >= 0" class="balance positive">
        Total savings this month: ₹{{ totalSavingsThisMonth.toLocaleString('en-IN') }}
      </h3>
      <h3 v-else class="balance negative">
        Over budget this month by ₹{{ Math.abs(totalSavingsThisMonth).toLocaleString('en-IN') }}
      </h3>
      </div>
      <h2>Total spent this month: ₹{{ totalSpentThisMonth.toLocaleString('en-IN') }}</h2>
      <h1 class="year-expense">Total Spent This Year: ₹{{ totalSpentThisYear.toLocaleString('en-IN') }}</h1>
    </div>
    
    <hr />
    <h3>Expenses:</h3>
    
    <ul class="expense-list">
      <li v-for="(expense, index) in sortedExpenses" :key="index" class="expense-item">
        <span>{{ expense.name }} - ₹{{ expense.amount.toLocaleString('en-IN') }} ({{ expense.date }})</span>
        <button @click="deleteExpense(index)" class="delete-btn">X</button>
      </li>
    </ul>

    <button @click="clearAllData" class="clear-btn">Clear All Data</button>
  </div>
</template>

<script>
export default {
  data() {
    return {
      newExpense: { name: '', amount: '', date: '' },
      expenses: JSON.parse(localStorage.getItem('expenses')) || [],
      errorMessage: '',
      budgetLimit: JSON.parse(localStorage.getItem('budgetLimit')) || 500,
    };
  },

  computed: {
    totalSpentToday() {
      const todayStr = this.getISTDate();
      return this.expenses
        .filter((e) => e.date === todayStr)
        .reduce((sum, e) => sum + e.amount, 0);
    },

    totalSpentThisMonth() {
      const { month, year } = this.getISTDateObj();
      return this.expenses
        .filter((e) => {
          const expenseDate = new Date(e.date);
          return expenseDate.getMonth() === month && expenseDate.getFullYear() === year;
        })
        .reduce((sum, e) => sum + e.amount, 0);
    },

    totalSpentThisYear() {
      const { year } = this.getISTDateObj();
      return this.expenses
        .filter((e) => new Date(e.date).getFullYear() === year)
        .reduce((sum, e) => sum + e.amount, 0);
    },

    remainingBalance() {
      return this.budgetLimit - this.totalSpentToday;
    },

    totalSavingsThisMonth() {
      const { month, year } = this.getISTDateObj();
      const dailySavings = {};

      this.expenses.forEach((expense) => {
        const expenseDate = new Date(expense.date);
        if (expenseDate.getMonth() === month && expenseDate.getFullYear() === year) {
          const dateKey = expense.date;
          if (!dailySavings[dateKey]) {
            dailySavings[dateKey] = this.budgetLimit;
          }
          dailySavings[dateKey] -= expense.amount;
        }
      });

      return Object.values(dailySavings).reduce((total, dailyBalance) => total + dailyBalance, 0);
    },

    sortedExpenses() {
      return [...this.expenses].sort((a, b) => new Date(b.date) - new Date(a.date));
    },
  },

  methods: {
    addExpense() {
      this.errorMessage = '';

      if (!this.newExpense.name || this.newExpense.amount === '' || !this.newExpense.date) {
        this.errorMessage = 'All fields are required';
        return;
      }

      if (this.newExpense.amount < 0) {
        this.errorMessage = 'Amount cannot be negative.';
        return;
      }

      const todayStr = this.getISTDate();

      if (this.newExpense.date > todayStr) {
        this.errorMessage = "Expenses for future dates can't be added.";
        return;
      }

      this.expenses.push({
        name: this.newExpense.name,
        amount: parseFloat(this.newExpense.amount),
        date: this.newExpense.date,
      });

      this.newExpense = { name: '', amount: '', date: '' };

      this.saveExpenses();
    },

    deleteExpense(index) {
      this.expenses.splice(index, 1);
      this.saveExpenses();
    },

    saveExpenses() {
      localStorage.setItem('expenses', JSON.stringify(this.expenses));
    },

    validateBudget() {
      if (this.budgetLimit < 0) {
        this.budgetLimit = 0;
      }
      localStorage.setItem('budgetLimit', JSON.stringify(this.budgetLimit));
    },

    clearAllData() {
      if (confirm("Are you sure you want to delete all data? This cannot be undone!")) {
        localStorage.removeItem('expenses');
        localStorage.removeItem('budgetLimit');
        this.expenses = [];
        this.budgetLimit = 150;
      }
    },

    getISTDate() {
      const now = new Date();
      const istOffset = 5.5 * 60 * 60 * 1000;
      const istDate = new Date(now.getTime() + istOffset);
      return istDate.toISOString().split('T')[0];
    },

    getISTDateObj() {
      const now = new Date();
      const istOffset = 5.5 * 60 * 60 * 1000;
      const istDate = new Date(now.getTime() + istOffset);
      return { month: istDate.getMonth(), year: istDate.getFullYear() };
    },
  },
};
</script>

<style scoped>
.app-container {
  
  margin: auto;
  text-align: center;
  background: linear-gradient(to right, #ECE9E6, #FFFFFF);
  min-height: 100vh;
  padding: 5px;
  padding-top: 1px;
  border-radius: 5px;
}
.title {
  font-size: 24px;
  font-weight: bold;
}
.budget-container {
  margin-bottom: 10px;
}
.input-container {
  display: flex;
  flex-direction: column;
  gap: 10px;
  margin-bottom: 20px;
}
.input-field-budget {
  padding: 10px;
  font-size: 16px;
  border: 1px solid #ccc;
  border-radius: 5px;
  width: 10%;
}
.input-field {
  padding: 10px;
  font-size: 16px;
  border: 1px solid #ccc;
  border-radius: 5px;
}
.date-container {
  display: flex;
  align-items: center;
  justify-content: space-between;
  width: 100%;
}

.date-label {
  flex: 0.5;
  font-weight: bold;
}

.date-input {
  flex: 1.5;
  padding: 8px;
  border: 1px solid #ccc;
  border-radius: 5px;
}

.add-btn {
  background-color: #28a745;
  color: white;
  padding: 10px;
  border: none;
  cursor: pointer;
  font-size: 16px;
  border-radius: 5px;
}
.add-btn:hover {
  background-color: #218838;
}
.error {
  color: red;
  font-size: 14px;
}
.expense-list {
  list-style: none;
  padding: 0;
}
.expense-item {
  display: flex;
  justify-content: space-between;
  background: #f8f9fa;
  padding: 10px;
  margin: 5px 0;
  border-radius: 5px;
}
.delete-btn {
  background: rgb(246, 103, 103);
  color: rgb(255, 255, 255);
  border: none;
  cursor: pointer;
  padding: 5px;
  border-radius: 20%;
}
.delete-btn:hover {
  background-color: rgb(203, 5, 5);
}
.summary {
  margin-top: 20px;
}
.balance {
  font-size: 18px;
  font-weight: bold;
}
.positive {
  color: green;
}
.negative {
  color: red;
}
.clear-btn {
  background-color: red;
  color: white;
  padding: 10px;
  border: none;
  cursor: pointer;
  font-size: 16px;
  border-radius: 5px;
  margin-top: 20px;
  display: block;
  width: 100%;
}
.clear-btn:hover {
  background-color: darkred;
}
.year-expense {
  padding: 8px;
  border-radius: 10px;
  background-color: #ECE9E6;
}
.savings {
  background-color: #f1ebeb;
  border-radius: 10px;
  padding: 1px;
}
</style>
