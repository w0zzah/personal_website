---  
title: "Interacting with a remote database"  
date: 2026-09-1
summary: "Learnt how to link a remote database to an application running Java for a seng202 project at the University of Canterbury"
---  

## What I learnt:
- How to establish a connection to a remote database in Java
- How to create a database schema using SQL (for mariadb)
- How to alter a remote table in Java using 'statement'.executeQuery()
- How to write exceptions for when the database fails


While we had been given a small amount of resources on how to setup the database, there was nothing really set in stone. I started with asking AI on how to set it up, but that led me down a rabbit hole of local database's, database types etc. So I decided to start with first connecting to it.


## Step 1: Connecting to the database.

When scrolling through the AI generated code I found the [DriverManager](https://docs.oracle.com/javase/8/docs/api/java/sql/DriverManager.html#getConnection-java.lang.String-) docs, specifically the function getConnection.
```java
public static Connection getConnection() throws SQLException {
    return DriverManager.getConnection(URL, USER, PASSWORD);
}
```
Once I had this setup with the url, username, and password all in plaintext, I committed and pushed to my remote branch! Little did I know at the time that this was something that would come back to bite me in the butt.


## Step 2: Creating the Database

After happily adding private information to the main repo, I faced the problem of actually interacting with the database. This is were my handy info of SQL came in clutch! (it did not, claude was the carry here.) However after reading through code I found it rather easy to understand on a surface level and pioneered through.

```sql
CREATE TABLE `users` (
  `id` INT(11) NOT NULL AUTO_INCREMENT,
  `username` VARCHAR(50) NOT NULL,
  `password` VARCHAR(255) NOT NULL,
  `user_type` VARCHAR(50) NOT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `idx_username` (`username`)
)
```

## Step 3: Adding to the Database (Back to java!)

Now I finally had everything setup to alter the table, but still couldn't figure out how to actually interact with it. I knew that I had to setup a SQL query in Java and sent it to the database but after the first sentence I was stunned.
```java
String query = "INSERT INTO users (username, password, user_type) VALUES (?, ?, ?) ";
```

Then after looking back at the docs I found that I can use a [Connection](https://docs.oracle.com/javase/8/docs/api/java/sql/Connection.html) to send the statement to the database
```java
try (Connection connection = DatabaseConnection.getConnection();
     PreparedStatement statement = connection.prepareStatement(query)) {
```

This set things in motion as I used string formatting to pass in the values required. (Note: at this point I was using jBCrypt to encrypt the passwords.)

```java
	statement.setString(1, username);
	statement.setString(2, PasswordService.hashPassword(password)); 
	statement.setString(3, userType.name());
```

But I still didn't know how to check if it was successful. So, it was back to Claudino for the finishing touches.
```java
    return statement.executeUpdate() > 0;
```
## Step 4: Checking the Database

After my first hurrah, everything else came easy.  Below is the code used to authenticate a user.

```java
public User authenticateUser(String username, String password) { 
	String query = """ SELECT id, username, password, user_type FROM users WHERE 
	username = ? """; 
		try (Connection connection = DatabaseConnection.getConnection(); 
			PreparedStatement statement = connection.prepareStatement(query)) { 
			statement.setString(1, username);
			try (ResultSet resultSet = statement.executeQuery()) {
				
				if (!resultSet.next()) {
					throw new InvalidCredentialsException();
				} 
				
				String storedHash = resultSet.getString("password");
				if (!PasswordService.checkPassword(password, storedHash)) {
					throw new InvalidCredentialsException();
				}
				UserType type;
				try {
					type = UserType.valueOf(resultSet.getString("user_type").toUpperCase());
				} catch (IllegalArgumentException e) {
						throw new DataAccessException("Stored user_type could not be parsed", e);
				} 
				return new User(
					resultSet.getInt("id"),
					resultSet.getString("username"),
					type); 
			}
		 } catch (SQLException e) {
			 throw new DataAccessException("Failed to authenticate user", e); 
		 }
	 }
```

## Mistakes + Hindsight

After feeling good for being the first group to implement a remote database, I was on sky 9. However that drastically changed when I found out about what a ".env" file was :/ Next time I would **NOT** commit private/hidden information to the remote repo, and analyse the required database schema before trying to create tables for it. 
