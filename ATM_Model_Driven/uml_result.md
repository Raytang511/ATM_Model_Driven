【类图】

- ATMController
  - 属性:
    - atmId: String
    - atmLocation: String
    - loggedInUser: User
  - 方法:
    - authenticateUser(cardNumber: String, pin: String): Boolean
    - getAccountBalance(accountId: String): Double
    - withdrawCash(accountId: String, amount: Double): Boolean
    - depositCash(accountId: String, amount: Double): Boolean
    - transferFunds(fromAccountId: String, toAccountId: String, amount: Double): Boolean
    - changePassword(accountId: String, oldPassword: String, newPassword: String): Boolean
    - dispenseCash(amount: Double): Void

- User
  - 属性:
    - userId: String
    - name: String
    - accounts: List<BankAccount>
  - 方法:
    - verifyPassword(pin: String): Boolean
    - getAccount(accountId: String): BankAccount

- BankAccount
  - 属性:
    - accountId: String
    - balance: Double
    - transactionHistory: List<Transaction>
  - 方法:
    - getBalance(): Double
    - deductAmount(amount: Double): Boolean
    - addAmount(amount: Double): Boolean
    - addTransaction(transaction: Transaction): Void

- Transaction
  - 属性:
    - transactionId: String
    - date: Date
    - amount: Double
    - accountId: String
    - transactionType: String
  - 方法:
    - createTransaction(accountId: String, amount: Double, transactionType: String): Transaction
    - getTransactionDetails(transactionId: String): Transaction

【顺序图】

《用户登录操作》

1. User sends authenticateUser(cardNumber, pin) request to ATMController.
   - ATMController verifies User's credentials internally.
   - ATMController updates loggedInUser attribute if credentials are valid.

2. ATMController responds with success/failure status to User.

《查询余额操作》

1. User sends getAccountBalance(accountId) request to ATMController.
   - ATMController checks if User is logged in.

2. ATMController sends getAccount(accountId) to User to fetch BankAccount.
   - User responds with BankAccount instance.

3. ATMController sends getBalance() to BankAccount.
   - BankAccount responds with current balance.

4. ATMController responds with balance to User.

《存款操作》

1. User sends depositCash(accountId, amount) request to ATMController.
   - ATMController checks if User is logged in.

2. ATMController sends verifyDeposit(amount) to ATM machine.
   - ATM machine verifies and confirms receipt of the cash.

3. ATMController sends getAccount(accountId) to User to fetch BankAccount.
   - User responds with BankAccount instance.

4. ATMController sends addAmount(amount) to BankAccount.
   - BankAccount adjusts balance and responds with success/failure status.

5. ATMController sends createTransaction(accountId, amount, "deposit") to Transaction.
   - Transaction logs the deposit and updates BankAccount's transactionHistory.

6. ATMController responds with success/failure status to User.

《取款操作》

1. User sends withdrawCash(accountId, amount) request to ATMController.
   - ATMController checks if User is logged in.

2. ATMController sends getAccount(accountId) to User to fetch BankAccount.
   - User responds with BankAccount instance.

3. ATMController sends getBalance() to BankAccount to check balance.
   - BankAccount responds with current balance.

4. ATMController verifies if sufficient funds are available.

5. If funds are sufficient, ATMController sends deductAmount(amount) to BankAccount.
   - BankAccount modifies balance and responds with success/failure status.

6. ATMController sends createTransaction(accountId, amount, "withdrawal") to Transaction.
   - Transaction logs the withdrawal and updates BankAccount's transactionHistory.

7. ATMController sends dispenseCash(amount) to ATM machine.
   - ATM machine dispenses cash and confirms action.

8. ATMController responds with success/failure status to User.

《转账操作》

1. User sends transferFunds(fromAccountId, toAccountId, amount) request to ATMController.
   - ATMController checks if User is logged in.

2. ATMController sends getAccount(fromAccountId) and getAccount(toAccountId) to User to fetch BankAccounts.
   - User responds with BankAccount instances for both accounts.

3. ATMController sends getBalance() to BankAccount(fromAccount) to check balance.
   - BankAccount responds with current balance.

4. ATMController verifies if sufficient funds are available.

5. If sufficient funds exist, ATMController sends deductAmount(amount) to BankAccount (fromAccount) and addAmount(amount) to BankAccount (toAccount).
   - Both BankAccount instances modify balances and respond with success/failure status.

6. ATMController sends createTransaction(fromAccountId, amount, "transfer") to Transaction.
   - Transaction logs the transfer for both accounts and updates their transactionHistory.

7. ATMController responds with success/failure status to User.

《修改密码操作》

1. User sends changePassword(accountId, oldPassword, newPassword) request to ATMController.
   - ATMController checks if User is logged in.

2. ATMController sends verifyPassword(oldPassword) to User.
   - User responds with success/failure to ATMController.

3. If successful, ATMController updates password and records event in log.

4. ATMController responds with success/failure status to User.