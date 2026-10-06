/*
    ===========================================================
     BANK MANAGEMENT APPLICATION (Console-based, C++)
    ===========================================================
    Concepts demonstrated:
      - Object-Oriented Programming (class, encapsulation,
        constructors, member functions)
      - File Handling (fstream, binary read/write) for
        persistent, secure storage of customer records

    Features:
      1. Open New Account
      2. Deposit Amount
      3. Withdraw Amount
      4. Balance Enquiry
      5. Display All Accounts
      6. Modify Account Details
      7. Close Account
      8. Exit

    Data is stored persistently in a binary file "accounts.dat".
    Each account is protected with a 4-digit PIN that must be
    entered correctly before deposit, withdrawal, balance
    enquiry, modification, or account closure.
    ===========================================================
*/

#include <iostream>
#include <fstream>
#include <iomanip>
#include <cstring>
#include <limits>

using namespace std;

class Account {
private:
    int accNo;
    char name[50];
    char pin[5];        // 4-digit PIN + null terminator
    char accType[15];   // "Savings" or "Current"
    double balance;

public:
    void openAccount(int newAccNo) {
        accNo = newAccNo;

        cout << "Enter Name: ";
        cin.getline(name, 50);

        cout << "Set a 4-digit PIN: ";
        cin.getline(pin, 5);

        cout << "Enter Account Type (Savings/Current): ";
        cin.getline(accType, 15);

        cout << "Enter Initial Deposit Amount: Rs. ";
        while (!(cin >> balance) || balance < 0) {
            cout << "Invalid amount. Enter a valid initial deposit: Rs. ";
            cin.clear();
            cin.ignore(numeric_limits<streamsize>::max(), '\n');
        }
        cin.ignore(numeric_limits<streamsize>::max(), '\n');
    }

    int getAccNo() const { return accNo; }
    string getName() const { return name; }
    double getBalance() const { return balance; }
    string getAccType() const { return accType; }

    bool verifyPin(const char* enteredPin) const {
        return strcmp(pin, enteredPin) == 0;
    }

    void deposit(double amount) {
        balance += amount;
    }

    bool withdraw(double amount) {
        if (amount > balance) {
            return false; // insufficient balance
        }
        balance -= amount;
        return true;
    }

    void modify() {
        cout << "Enter New Name: ";
        cin.getline(name, 50);
        cout << "Enter New Account Type (Savings/Current): ";
        cin.getline(accType, 15);
    }

    void display() const {
        cout << left << setw(10) << accNo
             << setw(20) << name
             << setw(12) << accType
             << fixed << setprecision(2) << setw(12) << balance << "\n";
    }

    void printSlip() const {
        cout << "\n--------------------------------\n";
        cout << "Account No : " << accNo << "\n";
        cout << "Name       : " << name << "\n";
        cout << "Type       : " << accType << "\n";
        cout << fixed << setprecision(2);
        cout << "Balance    : Rs. " << balance << "\n";
        cout << "--------------------------------\n";
    }
};

const char* FILENAME = "accounts.dat";

void clearInputBuffer() {
    cin.clear();
    cin.ignore(numeric_limits<streamsize>::max(), '\n');
}

void pauseScreen() {
    cout << "\nPress Enter to continue...";
    cin.get();
}

int getNextAccountNumber() {
    ifstream fin(FILENAME, ios::binary);
    int count = 0;
    Account acc;
    while (fin.read(reinterpret_cast<char*>(&acc), sizeof(Account))) {
        count++;
    }
    fin.close();
    return 1001 + count;
}

void readPin(char* buffer) {
    cout << "Enter 4-digit PIN: ";
    cin.getline(buffer, 5);
}

bool findAccount(int accNo, Account& acc, streampos& position) {
    ifstream fin(FILENAME, ios::binary);
    if (!fin) return false;

    while (fin.read(reinterpret_cast<char*>(&acc), sizeof(Account))) {
        if (acc.getAccNo() == accNo) {
            position = fin.tellg();
            position -= static_cast<streamoff>(sizeof(Account));
            fin.close();
            return true;
        }
    }
    fin.close();
    return false;
}

void openNewAccount() {
    Account acc;
    int newAccNo = getNextAccountNumber();

    cout << "\n----- Open New Account -----\n";
    cout << "Your Account Number will be: " << newAccNo << "\n";
    acc.openAccount(newAccNo);

    ofstream fout(FILENAME, ios::binary | ios::app);
    if (!fout) {
        cout << "Error opening file for writing.\n";
        return;
    }
    fout.write(reinterpret_cast<char*>(&acc), sizeof(Account));
    fout.close();

    cout << "\nAccount created successfully!\n";
    acc.printSlip();
    pauseScreen();
}

void depositAmount() {
    int accNo;
    cout << "\nEnter Account Number: ";
    while (!(cin >> accNo)) {
        cout << "Invalid input. Enter numeric Account Number: ";
        clearInputBuffer();
    }
    clearInputBuffer();

    Account acc;
    streampos pos;
    if (!findAccount(accNo, acc, pos)) {
        cout << "\nAccount not found.\n";
        pauseScreen();
        return;
    }

    char enteredPin[5];
    readPin(enteredPin);
    if (!acc.verifyPin(enteredPin)) {
        cout << "\nIncorrect PIN. Access denied.\n";
        pauseScreen();
        return;
    }

    double amount;
    cout << "Enter Amount to Deposit: Rs. ";
    while (!(cin >> amount) || amount <= 0) {
        cout << "Invalid amount. Enter a positive amount: Rs. ";
        clearInputBuffer();
    }
    clearInputBuffer();

    acc.deposit(amount);

    fstream file(FILENAME, ios::binary | ios::in | ios::out);
    file.seekp(pos);
    file.write(reinterpret_cast<char*>(&acc), sizeof(Account));
    file.close();

    cout << "\nDeposit successful!\n";
    acc.printSlip();
    pauseScreen();
}

void withdrawAmount() {
    int accNo;
    cout << "\nEnter Account Number: ";
    while (!(cin >> accNo)) {
        cout << "Invalid input. Enter numeric Account Number: ";
        clearInputBuffer();
    }
    clearInputBuffer();

    Account acc;
    streampos pos;
    if (!findAccount(accNo, acc, pos)) {
        cout << "\nAccount not found.\n";
        pauseScreen();
        return;
    }

    char enteredPin[5];
    readPin(enteredPin);
    if (!acc.verifyPin(enteredPin)) {
        cout << "\nIncorrect PIN. Access denied.\n";
        pauseScreen();
        return;
    }

    double amount;
    cout << "Enter Amount to Withdraw: Rs. ";
    while (!(cin >> amount) || amount <= 0) {
        cout << "Invalid amount. Enter a positive amount: Rs. ";
        clearInputBuffer();
    }
    clearInputBuffer();

    if (!acc.withdraw(amount)) {
        cout << "\nInsufficient balance! Current balance: Rs. "
             << fixed << setprecision(2) << acc.getBalance() << "\n";
        pauseScreen();
        return;
    }

    fstream file(FILENAME, ios::binary | ios::in | ios::out);
    file.seekp(pos);
    file.write(reinterpret_cast<char*>(&acc), sizeof(Account));
    file.close();

    cout << "\nWithdrawal successful!\n";
    acc.printSlip();
    pauseScreen();
}

void balanceEnquiry() {
    int accNo;
    cout << "\nEnter Account Number: ";
    while (!(cin >> accNo)) {
        cout << "Invalid input. Enter numeric Account Number: ";
        clearInputBuffer();
    }
    clearInputBuffer();

    Account acc;
    streampos pos;
    if (!findAccount(accNo, acc, pos)) {
        cout << "\nAccount not found.\n";
        pauseScreen();
        return;
    }

    char enteredPin[5];
    readPin(enteredPin);
    if (!acc.verifyPin(enteredPin)) {
        cout << "\nIncorrect PIN. Access denied.\n";
        pauseScreen();
        return;
    }

    acc.printSlip();
    pauseScreen();
}

void displayAllAccounts() {
    ifstream fin(FILENAME, ios::binary);
    if (!fin) {
        cout << "\nNo accounts found. File does not exist yet.\n";
        pauseScreen();
        return;
    }

    Account acc;
    bool found = false;

    cout << "\n----------------------------------------------------------------\n";
    cout << left << setw(10) << "Acc No" << setw(20) << "Name"
         << setw(12) << "Type" << setw(12) << "Balance" << "\n";
    cout << "----------------------------------------------------------------\n";

    while (fin.read(reinterpret_cast<char*>(&acc), sizeof(Account))) {
        found = true;
        acc.display();
    }
    fin.close();

    if (!found) {
        cout << "No account records available.\n";
    }
    cout << "----------------------------------------------------------------\n";
    pauseScreen();
}

void modifyAccount() {
    int accNo;
    cout << "\nEnter Account Number: ";
    while (!(cin >> accNo)) {
        cout << "Invalid input. Enter numeric Account Number: ";
        clearInputBuffer();
    }
    clearInputBuffer();

    Account acc;
    streampos pos;
    if (!findAccount(accNo, acc, pos)) {
        cout << "\nAccount not found.\n";
        pauseScreen();
        return;
    }

    char enteredPin[5];
    readPin(enteredPin);
    if (!acc.verifyPin(enteredPin)) {
        cout << "\nIncorrect PIN. Access denied.\n";
        pauseScreen();
        return;
    }

    cout << "\nCurrent Details:\n";
    acc.printSlip();
    cout << "\nEnter Updated Details:\n";
    acc.modify();

    fstream file(FILENAME, ios::binary | ios::in | ios::out);
    file.seekp(pos);
    file.write(reinterpret_cast<char*>(&acc), sizeof(Account));
    file.close();

    cout << "\nAccount details updated successfully!\n";
    pauseScreen();
}

void closeAccount() {
    int accNo;
    cout << "\nEnter Account Number to close: ";
    while (!(cin >> accNo)) {
        cout << "Invalid input. Enter numeric Account Number: ";
        clearInputBuffer();
    }
    clearInputBuffer();

    Account acc;
    streampos pos;
    if (!findAccount(accNo, acc, pos)) {
        cout << "\nAccount not found.\n";
        pauseScreen();
        return;
    }

    char enteredPin[5];
    readPin(enteredPin);
    if (!acc.verifyPin(enteredPin)) {
        cout << "\nIncorrect PIN. Access denied.\n";
        pauseScreen();
        return;
    }

    cout << "\nAre you sure you want to close this account? (y/n): ";
    char confirm;
    cin >> confirm;
    clearInputBuffer();
    if (tolower(confirm) != 'y') {
        cout << "Account closure cancelled.\n";
        pauseScreen();
        return;
    }

    ifstream fin(FILENAME, ios::binary);
    ofstream fout("temp.dat", ios::binary);
    Account temp;
    while (fin.read(reinterpret_cast<char*>(&temp), sizeof(Account))) {
        if (temp.getAccNo() != accNo) {
            fout.write(reinterpret_cast<char*>(&temp), sizeof(Account));
        }
    }
    fin.close();
    fout.close();

    remove(FILENAME);
    rename("temp.dat", FILENAME);

    cout << "\nAccount closed successfully!\n";
    pauseScreen();
}

void showMenu() {
    cout << "\n==============================================\n";
    cout << "         BANK MANAGEMENT APPLICATION\n";
    cout << "==============================================\n";
    cout << "1. Open New Account\n";
    cout << "2. Deposit Amount\n";
    cout << "3. Withdraw Amount\n";
    cout << "4. Balance Enquiry\n";
    cout << "5. Display All Accounts\n";
    cout << "6. Modify Account Details\n";
    cout << "7. Close Account\n";
    cout << "8. Exit\n";
    cout << "==============================================\n";
    cout << "Enter your choice: ";
}

int main() {
    int choice;

    do {
        showMenu();
        while (!(cin >> choice)) {
            cout << "Invalid input. Enter a number between 1-8: ";
            clearInputBuffer();
        }
        clearInputBuffer();

        switch (choice) {
            case 1: openNewAccount(); break;
            case 2: depositAmount(); break;
            case 3: withdrawAmount(); break;
            case 4: balanceEnquiry(); break;
            case 5: displayAllAccounts(); break;
            case 6: modifyAccount(); break;
            case 7: closeAccount(); break;
            case 8: cout << "\nThank you for banking with us!\n"; break;
            default: cout << "\nInvalid choice. Please try again.\n"; pauseScreen();
        }

    } while (choice != 8);

    return 0;
}
