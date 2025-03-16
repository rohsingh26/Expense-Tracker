<template>
  <div class="app-container">
    <h1 class="title">Expense Tracker</h1>

    <!-- Budget Input -->
    <div class="budget-container">
      <label>Set Daily Budget: </label>
      <input
        type="number"
        v-model.number="budgetLimit"
        class="input-field-budget"
        @change="validateBudget"
      />
    </div>

    <!-- Expense Input Section -->
    <div class="input-container">
      <input type="text" v-model="newExpense.name" placeholder="Expense Name" class="input-field" />
      <input
        type="number"
        v-model.number="newExpense.amount"
        placeholder="Amount"
        class="input-field"
      />
      <input type="date" v-model="newExpense.date" class="input-field" />
      <button @click="addExpense" class="add-btn">Add Expense</button>
      <p v-if="errorMessage" class="error">{{ errorMessage }}</p>
    </div>

    <!-- Summary -->
    <div class="summary">
      <h3>Today's Expenses: ₹{{ totalSpentToday.toLocaleString('en-IN') }}</h3>
      <p v-if="remainingBalance > 0" class="balance positive">
        Remaining Balance: +₹{{ remainingBalance.toLocaleString('en-IN') }}
      </p>
      <p v-if="remainingBalance < 0" class="balance negative">
        Deficit: -₹{{ Math.abs(remainingBalance).toLocaleString('en-IN') }}
      </p>

      <h3 v-if="totalSavingsThisMonth >= 0" class="balance positive">
        Total Savings This Month: ₹{{ totalSavingsThisMonth.toLocaleString('en-IN') }}
      </h3>
      <h3 v-else class="balance negative">
        Over Budget This Month by ₹{{ Math.abs(totalSavingsThisMonth).toLocaleString('en-IN') }}
      </h3>

      <h2>Total Spent This Month: ₹{{ totalSpentThisMonth.toLocaleString('en-IN') }}</h2>
      <h1>Total Spent This Year: ₹{{ totalSpentThisYear.toLocaleString('en-IN') }}</h1>
    </div>

    <!-- Expenses List -->
    <ul class="expense-list">
      <li v-for="(expense, index) in sortedExpenses" :key="index" class="expense-item">
        <span>{{ expense.name }} - ₹{{ expense.amount.toLocaleString('en-IN') }} ({{ expense.date }})</span>
        <button @click="deleteExpense(index)" class="delete-btn">X</button>
      </li>
    </ul>

    <!-- Clear All Data Button -->
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
      budgetLimit: JSON.parse(localStorage.getItem('budgetLimit')) || 150,
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
  width: 85%;
  margin: auto;
  text-align: center;
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
  background: red;
  color: white;
  border: none;
  cursor: pointer;
  padding: 5px;
  border-radius: 50%;
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
</style>
