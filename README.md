# Inferno Bank 🏦

Proyecto de ejemplo con **microservicios en Node.js** y despliegue con **Terraform en AWS**.

## 📂 Estructura del proyecto

micro-services/
├── card-service/
├── notification-service/
└── user-service/

terraform/
├── main.tf
├── provider.tf
└── ...


Instrucciones del reto
├── Description
- A small business is planning to launch a digital banking platform and requires a robust,
scalable infrastructure capable of handling thousands of transactions concurrently.
Recognizing the need for high availability, security, and flexibility, they decided to
leverage AWS services to build and optimize their backend systems. By adopting a
microservices architecture, they aim to ensure modularity, easier maintenance, and the
ability to scale individual components independently. This approach positions them to
deliver reliable, high-performance banking services while remaining agile in a
competitive financial technology landscape.

├── Libraries (We are gonna use SDK V3 Because V2 is
already deprecated)
Javascript
• @aws-sdk/client-dynamodb
• @aws-sdk/lib-dynamodb
• @aws-sdk/client-s3
• @aws-sdk/s3-request-presigner
• @aws-sdk/client-sqs
• @aws-sdk/client-secrets-manager

├── Objective
Design and implement a banking system using Terraform to manage users, cards,
transactions, and notifications, with an event-driven architecture that leverages
messaging queues, and storage in DynamoDB and S3.

├── Microservices and functionalities
User services (user-service)
• It allows name, lastName, email, password(encrypted),documentNumber
• Login: Authentication (email + password) → return jwt
• Information update: It allows direction, phoheNumber
• Upload profile image: It allows base64 string, store in S3 and update dynamoDb with the url the images’ name must be create by lambda to avoid duplicate namesin s3

├── Component’s list

├── • Aws api gateway
◦ POST /register
◦ POST /login
◦ PUT /profile/{user_id}
◦ POST /profile{user_id}/avatar
◦ GET /profile/{user_id}

├── • Aws dynamodb
◦ user-table
▪ partition key: uuid
▪ order key: document

├── • Aws lambda
◦ register-user-lambda
◦ login-user-lambda
◦ update-user-lambda
◦ upload-avatar-user-lambda
◦ get-profile-user-lambda

├── • Aws Secrets manager
◦ password-secret

├── Contracts
POST /register (register-user-lambda)
POST /login (login-user-lambda)
PUT /profile/{user_id} (update-user-lambda)
POST /profile{user_id}/avatar (upload-avatar-user-lambda)
GET /profile/{user_id} (get-profile-user-lambda)

├──  Card services (card-service)
Meanwhile creating an user, it must generate a credit card request (via SQS)
A lambda (card-approval-worker) consume a queue and:
• Generate randomly a score from 0 to 100
• Assign the amount with the algorithm:
◦ amount = 100 + (score/100) ∗ (10000000 − 100)
• Store the request in database with status “PENDING” only credit card
• The debit card must be created with the status ACTIVATED

├── Component’s list

├── • Api gateway
◦ POST /card/activate
◦ POST /transactions/save/{card_id}
◦ POST /card/paid/{card_id}
◦ GET /card/{card_id}
├── • AWS Sqs
◦ create-request-card-sqs

├── • AWS Dlq
◦ error-create-request-card-sqs

├── • AWS Lambdas
◦ create-request-card-lambda
◦ card-activate-lambda
◦ card-purchase-lambda
◦ card-transaction-save-lambda
◦ card-paid-credit-card-lambda
◦ card-get-report-lambda
◦ card-request-failed (to manage failed error in sqs)

├──• AWS dynamodb
◦ card-table
▪ partition key: uuid
▪ order key: createdAt
◦ transaction-table
▪ partition key: uuid
▪ order key: createdAt
◦ card-table-error
▪ partition key: uuid
▪ order key: createdAt

├── • AWS S3
◦ trasnsactions-report-bucket (remember the name must be unique globally)

├── Contracts
▪ SQS (create-request-card-sqs) (create-request-card-lambda)
Create a debit card (It call automatically asynchronous when create the user)

▪ SQS (create-request-card-sqs) (create-request-card-lambda)
Create a credit card (It call automatically asynchronous when create the user)

▪ POST /card/activate (card-activate-lambda)
When the user completes 10 transactions update status in database to ACTIVATED

▪ POST /transactions/purchase (card-purchase-lambda)
If card is a debit card validate the balance in the account is enough to purchase if not
return an error
if it has enough money continue the transaction and balance - amount = new balance
to set in database
if it is a credit card check the balance used if the transaction exceeds the card limit if
exceeds the balance return an error if not continue with the transaction

▪ POST /transactions/save/{card_id} (card-transaction-save-lambda)
To add new balance to debit cards

▪ POST /card/paid/{card_id} (card-paid-credit-card-lambda)
To payment the credit card used balance

▪ GET /card/{card_id} (card-get-report-lambda)
to get the register of all transactions in a csv file or excel file and send data via email

├──  Notification service
Meanwhile user are using the app this service must be sending notifications to the user
via email, the service must be able to handle the multiple template of all the events in
the app

├──  Component’s list

├──  • SQS
◦ notification-email-sqs

├── • DQL
◦ notification-email-error-sqs

├── • AWS Lambda
◦ send-notifications-lambda
◦ send-notifications-error-lambda

├── • S3
◦ templates-email-notification

├── • AWS Dynamo
◦ notification-table
▪ primary key: uuid
▪ order key: createdAt
◦ notification-error-table
▪ primary key: uuid
▪ order key: createdAt

├── Contracts
SQS(notification-email-sqs)
Send email notification, It can handle the next notifications:
The template must be storage in S3 Bucket and get as demand the lambda need it
• WELCOME:
◦ Send email when user register inside the app
• USER.LOGIN
◦ Send email when user login the app, each login send email
• USER.UPDATE
◦ Send email when user update the information in the app
• CARD.CREATE
◦ Send Email when a card is created Credit Card
• CARD.ACTIVATE
◦ Send Email when credit card is activated
• TRANSACTION.PURCHASE
◦ Send email when user buy anything with a credit or debit card
• TRANSACTION.SAVE
◦ Send email when user add balance to debit card
• TRANSACTION.PAID
◦ Send email when user pay the credit card’s used balance
• REPORT.ACTIVITY
◦ Send email when user wants to know the transactions
(LAMBDA) send-notifications-lambda
