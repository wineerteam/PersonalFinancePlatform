# PersonalFinancePlatform

PersonalFinancePlatform
├── src
│   ├── Main.java                # Entry point of the application
│   ├── Database.java            # Handles database connection and queries
│   ├── AdminDashboard.java      # GUI for admin functionalities
│   ├── AdvisorDashboard.java    # GUI for financial advisors
│   ├── UserDashboard.java       # GUI for regular users
│   └── Models
│       ├── User.java            # Represents a user in the system
│       ├── Expense.java         # Represents an expense
│       └── Budget.java          # Represents a budget
├── libs
│   └── sqlite-jdbc-3.36.0.3.jar # SQLite JDBC driver
├── resources
│   ├── icons                    # Directory for GUI icons
│   └── css
│       └── theme.css            # Custom styling for the GUI
├── database
│   └── finance.db               # SQLite database file
├── launch.json                  # Launch configuration for development
└── README.md                    # Project documentation
