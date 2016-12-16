---
layout: post
title: Set up Elgg in Azure App Services
---
1. First we start a Web App in Azure App Services.
1. We create a Github repo with the Elgg framework and push it.
1. We set up the deployment source pointing to our repo.
1. Now if we visit the site we have the install message ready to go.

1. In the meantime we deploy a bitnami MySQL VM
1. We change the Mysql Pass
```
mysqladmin -u root -p'bitnami' password newpass
```
1. Usando MySQL Workbench nos conectamos a la DB y creamos un nuevo usuario
1. We create a new DB Schema with UTF8 collation
1. We add all the permissions to the new schema for the new user.
1. Next thing is to make sure you can connect with your new user to mysql
1. Now, lets go back to our original URL and see the process of the test that Elgg is performing on the server, you will get something like this:

1. Lets add a Web.config to solve the rewrite error in our project, after doing this we just make a commit and push it to the server.

```
git commit -m "Adding web.config file" 
git push origin master
``` 

1. We can check that the file has already being deployed by using kudu.
1. Just go to your url and add the string ".scm" after the name of your website 
1. In there you can just go to the Powershell and go to "D:\home\site\wwwroot>" where you can do an "ls" to see if the new file is there. You can also do "cat web.config" to see the contents of the file.

1. Now we refresh the test webpage and we shall see that the test was passed. We can proceed to the next step.
1. Now we need to fill the form with all the data of our new website. Also we need to add the Data Directory with an absolute path. So what I will do is to create an empty folder called "data" with file in it so it can be tracked by git.

```
mkdir data
echo "" >> data\text.txt
````

1. And then we commit it and push it

```
git add data\
git commit -m "Adding data folder"
git push
````
1. Now we can check the folder in powershell and obtain the absolute path. This is the one in my case:

```
D:\home\site\wwwroot\data
```

1. We proceed to create the admin account

1. And we are ready to start using our Elgg Website





