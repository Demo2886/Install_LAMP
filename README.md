## Install Linux, Apache, MySQL, PHP (LAMP) stack on Ubuntu 20.04

**Step 1 — Installing Apache and Updating the Firewall**

```bash
sudo apt update
sudo apt install apache2
sudo ufw allow 'Apache'
```

**Step 2 — Installing MySQL**

```bash
sudo apt install mysql-server
```
When the installation is finished, it’s recommended that you run a security script that comes pre-installed with MySQL. This script will remove some insecure default settings and lock down access to your database system. Start the interactive script by running:

```bash
sudo mysql_secure_installation
```
**Note:** *Enabling this feature is something of a judgment call. If enabled, passwords which don’t match the specified criteria will be rejected by MySQL with an error. It is safe to leave validation disabled, but you should always use strong, unique passwords for database credentials.*

**Step 3 — Installing PHP**

```bash
sudo apt install php php-mysql libapache2-mod-php
```
Once the installation is finished, you can run the following command to confirm your PHP version:

```bash
php -v
```

**Step 4 — Creating a Virtual Host for your Website**

Create the directory for **your_domain** as follows:

```bash
sudo mkdir /var/www/my_domain
``
Next, assign ownership of the directory with the $USER environment variable, which will reference your current system user:

```bash
sudo chown -R $USER:$USER /var/www/my_domain
```
Then, open a new configuration file in Apache’s sites-available directory using your preferred command-line editor. Here, we’ll use nano:

```bash
sudo nano /etc/apache2/sites-available/my_domain.conf
```
This will create a new blank file. Paste in the following bare-bones configuration:

/etc/apache2/sites-available/your_domain.conf

```apache
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/my_domain
    ServerName my_domain
    ServerAlias www.my_domain
    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
With this ```VirtualHost``` configuration, we’re telling Apache to serve **your_domain** using _/var/www/your_domain_** as the web root directory. If you’d like to test Apache without a domain name, you can remove or comment out the options ```ServerName``` and ```ServerAlias``` by adding a ```#``` character in the beginning of each option’s lines.

You can now use a2ensite to enable the new virtual host:

```bash
sudo a2ensite my_domain
```

You might want to disable the default website that comes installed with Apache. This is required if you’re not using a custom domain name, because in this case Apache’s default configuration would overwrite your virtual host. To disable Apache’s default website, type:

```bash
sudo a2dissite 000-default
```
To make sure your configuration file doesn’t contain syntax errors, run:

```bash
sudo apachectl configtest
```
Finally, reload Apache so these changes take effect:

```bash
sudo systemctl reload apache2
```
Your new website is now active, but the web root _/var/www/your_domain_** is still empty. Create an ```index.html``` file in that location so that we can test that the virtual host works as expected:

```bash
sudo touch /var/www/my_domain/index.html
```
Include the following content in this file:

```html
<html>
  <head>
    <title>your_domain website</title>
  </head>
  <body>
    <h1>Hello World!</h1>

    <p>This is the landing page of <strong>your_domain</strong>.</p>
  </body>
</html>
```
You can leave this file in place as a temporary landing page for your application until you set up an index.php file to replace it. Once you do that, remember to remove or rename the index.html file from your document root, as it would take precedence over an index.php file by default.


**Step 5 — Testing PHP Processing on your Web Server**

Now that you have a custom location to host your website’s files and folders, we’ll create a PHP test script to confirm that Apache is able to handle and process requests for PHP files.

Create a new file named **info.php** inside your custom web root folder:

```bash
sudo touch /var/www/my_domain/info.php
```
This will open a blank file. Add the following text, which is valid PHP code, inside the file:

```php
<?php
phpinfo(); // Display PHP information
?>
```
To test this script, go to your web browser and access your server’s domain name or IP address, followed by the script name, which in this case is **info.php**:

```bash
http://my_domain/info.php
```
After checking the relevant information about your PHP server through that page, it’s best to remove the file you created as it contains sensitive information about your PHP environment and your Ubuntu server. You can use ```rm``` to do so:

```bash
sudo rm /var/www/my_domain/info.php
```

**Step 6 — Testing Database Connection from PHP (Optional)**

If you want to test whether PHP is able to connect to MySQL and execute database queries, you can create a test table with dummy data and query for its contents from a PHP script. Before we can do that, we need to create a test database and a new MySQL user properly configured to access it.

At the time of this writing, the native MySQL PHP library mysqlnd doesn’t [support](https://www.php.net/manual/en/ref.pdo-mysql.php) ```caching_sha2_authentication```, the default authentication method for MySQL 8. We’ll need to create a new user with the ```mysql_native_password``` authentication method in order to be able to connect to the MySQL database from PHP.

We’ll create a database named **example_database** and a user named **example_user, but you can replace these names with different values.

First, connect to the MySQL console using the ```root``` account:

```bash
mysql -u root
```
To create a new database, run the following command from your MySQL console:

```bash
CREATE DATABASE example_database;
```
Now you can create a new user and grant them full privileges on the custom database you’ve just created.

The following command creates a new user named example_user, using mysql_native_password as default authentication method. We’re defining this user’s password as password, but you should replace this value with a secure password of your own choosing.

```bash
CREATE USER 'example_user'@'localhost' IDENTIFIED WITH mysql_native_password BY 'password';
```
Now we need to give this user permission over the **example_database** database:

```bash
GRANT ALL PRIVILEGES ON example_database.* TO 'example_user'@'localhost';
```
This will give the **example_user** user full privileges over the **example_database** database, while preventing this user from creating or modifying other databases on your server.

Now exit the MySQL shell with:

```bash
exit
```

You can test if the new user has the proper permissions by logging in to the MySQL console again, this time using the custom user credentials:

```bash
mysql -u example_user -p
```
Notice the ``-p`` flag in this command, which will prompt you for the password used when creating the ```example_user``` user. After logging in to the MySQL console, confirm that you have access to the ```example_database``` database:

```bash
SHOW DATABASES;
```
Next, we’ll create a test table named **todo_list**. From the MySQL console, run the following statement:

```bash
CREATE TABLE example_database.todo_list (
	item_id INT AUTO_INCREMENT,
	content VARCHAR(255),
	PRIMARY KEY(item_id)
);
```
Insert a few rows of content in the test table. You might want to repeat the next command a few times, using different values:

```bash
INSERT INTO example_database.todo_list (content) VALUES ("My first important item");
```

To confirm that the data was successfully saved to your table, run:

```bash
SELECT * FROM example_database.todo_list;
```
After confirming that you have valid data in your test table, you can exit the MySQL console:

```bash
exit
``` 
Now you can create the PHP script that will connect to MySQL and query for your content. Create a new PHP file in your custom web root directory using your preferred editor. We’ll use ``nano`` for that:

```bash
nano /var/www/your_domain/todo_list.php
```
The following PHP script connects to the MySQL database and queries for the content of the todo_list table, exhibiting the results in a list. If there’s a problem with the database connection, it will throw an exception. Copy this content into your ```todo_list.php``` script:

```php
<?php
$user = "example_user";
$password = "password";
$database = "example_database";
$table = "todo_list";

try {
  $db = new PDO("mysql:host=localhost;dbname=$database", $user, $password);
  echo "<h2>TODO</h2><ol>"; 
  foreach($db->query("SELECT content FROM $table") as $row) {
    echo "<li>" . $row['content'] . "</li>";
  }
  echo "</ol>";
} catch (PDOException $e) {
    print "Error!: " . $e->getMessage() . "<br/>";
    die();
}
```
Save and close the file when you’re done editing.
You can now access this page in your web browser by visiting the domain name or public IP address configured for your website, followed by _/todo_list.php_**:

```bash
http://your_domain/todo_list.php
```

**Conclusion**
In this guide, we’ve built a flexible foundation for serving PHP websites and applications to your visitors, using Apache as web server and MySQL as database system.

As an immediate next step, you should ensure that connections to your web server are secured, by serving them via HTTPS. In order to accomplish that, you can use [Let’s Encrypt](https://www.digitalocean.com/community/tutorials/how-to-secure-apache-with-let-s-encrypt-on-ubuntu-20-04) to secure your site with a free TLS/SSL certificate.
