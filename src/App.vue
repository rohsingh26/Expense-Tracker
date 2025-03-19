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
    
    <h3>
      <span @click="toggleExpenses" class="toggle-arrow">
        {{ showAll ? 'Expenses: ▲' : 'Expenses: ▼' }}
      </span>
    </h3>

    <ul class="expense-list">
      <li v-for="(expense, index) in displayedExpenses" :key="index" class="expense-item">
        <span>{{ expense.name }} - ₹{{ expense.amount.toLocaleString('en-IN') }} ({{ expense.date }})</span>
        <button @click="deleteExpense(index)" class="delete-btn">X</button>
      </li>
    </ul>

    <div v-if="sortedExpenses.length > 2 && !showAll" class="dots" @click="toggleExpenses">...</div>

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
      showAll: false
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

    displayedExpenses() {
      return this.showAll ? this.sortedExpenses : this.sortedExpenses.slice(0, 2);
    }
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

      this.expenses.unshift({
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

    toggleExpenses() {
      this.showAll = !this.showAll;
    }
  }
};
</script>
