# MariaDB

### Troubleshooting

#### MySQLi "No such file or directory" with Docker Database

##### Situation
- MariaDB/MySQL running in Docker container
- Port 3306 exposed to host (`0.0.0.0:3306->3306/tcp`)
- PHP application using mysqli driver (e.g., via Dibi/Doctrine)
- Database connection configured with `host: 'localhost'`

##### Symptom
```
Fatal error: Uncaught Dibi\DriverException: No such file or directory
in .../MySqliDriver.php:102
```

The error occurs during database connection, even though:
- The PHP file exists
- mysqli extension is installed
- Docker container is running
- Port 3306 is accessible

##### Root Cause
When PHP's mysqli sees `host: 'localhost'`, it attempts a **Unix socket connection** and looks for a socket file (e.g., `/var/run/mysqld/mysqld.sock`). Since the database runs in Docker, no socket file exists on the host filesystem—only the TCP port is exposed.

##### Solution
Change database host from `localhost` to `127.0.0.1` to force TCP connection:

```php
const CONFIG = [
    'database' => [
        'driver' => 'mysqli',
        'host' => '127.0.0.1',  // ← Use IP, not 'localhost'
        'username' => 'root',
        'password' => 'root',
        'database' => 'testdb',
    ],
];
```

##### Why This Works
- `localhost` → mysqli uses Unix socket
- `127.0.0.1` → mysqli uses TCP/IP connection

Docker only exposes TCP ports, not Unix sockets.

##### Additional Notes
- This applies to any PHP mysqli connection to containerized databases
- Also relevant for PDO MySQL connections with default settings
- Alternative: Use `127.0.0.1` or actual container IP in all database configs
