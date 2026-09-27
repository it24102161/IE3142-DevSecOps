\# Member 1 – Application, Architecture and Containerisation



\## 1. Student Details



Student ID: IT24102161  

Name: Kadiman M G



\## 2. Selected Application



OWASP Juice Shop



\## 3. Reason for Selection



OWASP Juice Shop is an open-source deliberately insecure web

application suitable for security testing and DevSecOps practice.



\## 4. Technology Stack



\- Angular

\- Node.js

\- Express

\- SQLite

\- Docker



\## 5. Application Setup



The application was cloned from the official OWASP Juice Shop

repository and tested locally using npm.



\## 6. Application Architecture



The browser communicates with the Juice Shop application,

which contains the Angular frontend and Node.js/Express

application logic and uses SQLite for data storage.



\## 7. Containerisation



The existing Dockerfile supplied by the Juice Shop project

was used to build the application container.



\## 8. Docker Compose



A project-level docker-compose.yml was created so that the

application can be built and started using one command.



\## 9. Trust Boundary



The primary trust boundary is between the user's web browser

and the application running inside the Docker container.



\## 10. Evidence



Screenshots demonstrate local execution, Docker image creation,

Docker container execution and Docker Compose execution.

