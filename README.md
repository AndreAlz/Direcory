# Directory

## Introduction
This app was built to control the differents *annexes* we have in a contact center, improving their **search**, **control** and **management**.
To understand how Shiny works in general, you have to understand the concept of a different paradigm, **reactive programming**.

> Reactive programming is a subset of asynchronous programming and a paradigm where the availability of new information drives the logic forward rather than having control flow driven by a thread-of-execution.
>- Jonas Bonér and Viktor Klang, [Reactive Programming vs. Reactive Systems](https://www.oreilly.com/ideas/reactive-programming-vs-reactive-systems)

The reactivity and data flow of this repository is showed in this simple map:

![Reactivity Map](https://user-images.githubusercontent.com/31576039/45839122-11f51680-bcd9-11e8-9e41-f906d103aa4f.png)

This is an example of how the Shiny app looks like in the end:

![Overview](https://user-images.githubusercontent.com/31576039/45829616-f847d500-bcc0-11e8-9883-ce55a82eb261.png)

## Libraries
I am using the following libraries in *server.R*:

Library | Usage | Documentation
------- | ----- | -------------
DBI | Connection to the DB, sending queries and catching resultant information | [DBI package R](https://www.rdocumentation.org/packages/DBI/versions/0.5-1)
RMySQL | Driver for the connection | [RMySQL package R](https://www.rdocumentation.org/packages/RMySQL/versions/0.10.15)
DT | Select row in the datatable rendered by Shiny | [DT package R](https://www.rdocumentation.org/packages/DT/versions/0.4)
data.table | Better management of the information uploaded in an Excel file | [data.table package R](https://www.rdocumentation.org/packages/data.table/versions/1.11.6)
readxl | Read and save the Excel file uploaded even though it is a 'xls' or 'xlsx' | [readxl package R](https://www.rdocumentation.org/packages/readxl/versions/1.1.0)
Stringr | Creating a query to insert all the information from the Excel(found it in [here](https://user-images.githubusercontent.com/31576039/45834019-c425e180-bccb-11e8-9321-04dc81f1d093.png)) | [Stringr package R](https://www.rdocumentation.org/packages/stringr/versions/1.3.1)

## Functions
We use three simple functions to create connection with the Database, simulate clicking a button with JS and avoiding all the existing data in the dataset uploaded on the page:

### Connection with the Database
```R
connectMySQL<-function(){
  m = dbDriver("MySQL")
  myHost <- "YOUR IPP"
  myUsername = "USER"
  myDbname = "DBNAME"
  myPort = "PORT"
  myPassword = "PASSWORD"
  con = dbConnect(m, user= myUsername, host= myHost, password= myPassword, dbname= myDbname, port= myPort)
  return(con)
  }
```

### Simulation of clicking a button
```javascript
jQuery(document).ready(function(){
  jQuery('#nombre').keypress(function(evt){
    if (evt.keyCode == 13){
      jQuery('#bnombre').click();
    }
  });
});
```

### Avoiding duplicates
```R
depurar<-function(fileb,conne){
  duplicados=matrix(nrow = 0, ncol = 6)
  conn=conne
  consultaquery="select anexo from catalogo"
  filei=t(as.numeric(unlist(fileb[,3])));
  i=1
  flag=0
  for (i in i:length(filei)) {
    resu = dbSendQuery(conn, consultaquery)
    sqlb = dbFetch(resu,n=-1)
    j=1
    for (j in j:length(sqlb[,1])) {
      if(toString(filei[1,i])==toString(sqlb[j,1])){
        duplicados=rbind(duplicados,fileb[i,])
        flag=1
        break
      }}
      if(flag==0){
        consulta1="insert into catalogo(nombre, campa, anexo, cel, sede,cargo) value ('"
        consulta2="', '"
        consulta3="')"
        query=paste(consulta1,
                            fileb[i,1],consulta2,fileb[i,2],
                            consulta2,fileb[i,3],consulta2,fileb[i,4],
                            consulta2,fileb[i,5],consulta2,fileb[i,6],
                            consulta3,sep = "")
        res=dbSendQuery(conn,query)
        dbClearResult(res)
      }else{
        flag=0
      }
    dbClearResult(resu)
  }
  return(duplicados)
}
```

## UI
The *ui.R* file it just used to render the different views in the webpage, as you can see in the following code:
```R
library(shiny)
shinyUI(bootstrapPage(
    uiOutput("login"),
    uiOutput("admin"),
    uiOutput("usr")
  )
)
```
As you can see there are four lines of code because we are rendering all the information and functions of the webpage in the *server.R* file.

## Server
In *server.R* you will find all the libraries to be used and the _*.R_ files with different funcions:
```R
library(shiny)
library(DBI)
library(RMySQL)
library(DT)
library(data.table)
library(readxl)
library(stringr)
source("conect.R")
source("depurar.R")
```
### Main Login
This function will be show as the first view for the user. Its important to say that the *actionButton called "entrar"* will be observe for **login admin** and **login user**.
```R
output$login=renderUI({
    fluidPage(
      includeScript("entre.js"),
      mainPanel(
        titlePanel("DIRECTORIO DE ANEXOS"),
        textInput("usuario", "USUARIO"),
        passwordInput("pw", "CLAVE"),
        actionButton("entrar", "ENTRAR")
      ))
  })
  ```
### Admin and User Login
This function react to the changes on the values of [Main Login](https://github.com/AndreAlz/Directory/blob/master/README.md#main-login)
  ```R
   observeEvent(input$entrar,{
    output$admin=renderUI({
      if (toString(input$usuario)=="admin" && toString(input$pw)=="mdy12345") {...}
      
   observeEvent(input$entrar,{
    output$usr=renderUI({
      if (toString(input$usuario)=="mdyuser" && toString(input$pw)=="12345") {...}
  ```
What we do is see if the user and the password are correct, if it so we would show differents views of the same web page with a different amount of functionalities. Actually, the admin user would have all the functionalities like **search**, **delete**, **insert** and **B¿bulk insert**, while the normal user just have the option to **search** and **show** the information.

#### Bulk Insertion
The function of bulk insertion is only available for the admin user as we said before, and for me is the most interesting function here we see its logic:
```R
observeEvent(input$afirmaM,{
    if (tabledata!=0) {
      tabledata[,1][is.na(tabledata[,1])]<-"--"
      tabledata[,2][is.na(tabledata[,2])]<-"--"
      tabledata[,4][is.na(tabledata[,4])]<-"--"
      tabledata[,5][is.na(tabledata[,5])]<-"--"
      tabledata[,6][is.na(tabledata[,6])]<-"--"
      conn=connectMySQL()
      duplicados=depurar(tabledata,conn)
      output$resultado=DT::renderDataTable({
        duplicados
      })
      tabledata<<-0
    }
  })
```
