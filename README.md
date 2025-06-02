# School Application Form Project

### Overview

This project is a full-stack web application designed to allow users to register for school programs. The application consists of a frontend hosted on AWS S3 and CloudFront, and a backend deployed on AWS Elastic Beanstalk (EB) with an RDS MySQL database. The backend integrates with AWS Secrets Manager and Parameter Store for secure credential management. Users can submit their name, email, phone number, and program of interest through a form, and the data is stored in the database. The application includes validation, error handling, and a user-friendly frontend experience.

---

## Project Structure

- **Frontend**:
  - Hosted on AWS S3 (`schoolapp.classof25.online`) with CloudFront for HTTPS.
  - `index.html`: A single-page application with a form for user registration, styled with CSS, and JavaScript for form submission and response handling.
- **Backend**:
  - Deployed on AWS Elastic Beanstalk (`schoolapp-backend` environment).
  - Built with Node.js and Express.
  - Uses MySQL (RDS) for data storage.
  - Integrates with AWS Secrets Manager for database credentials and Parameter Store for configuration (e.g., database host, name, user).
- **Database**:
  - AWS RDS MySQL instance (`ebsdatabase.******`).
  - Database: `dbName`.
  - Table: `users` (columns: `id`, `name`, `email`, `phone`, `program`, `created_at`).

1. **Backend Development**:
   - Created a Node.js/Express backend with the following files:
     - `index.js`: Main entry point, sets up the Express server and routes.
     - `routes/userRoutes.js`: Defines the `/api/register` endpoint.
     - `controllers/userController.js`: Handles user registration logic with input validation.
     - `models/userModel.js`: Manages database operations (e.g., creating the `users` table, inserting users).
     - `config/aws.js`: Handles AWS SDK integration for Secrets Manager and Parameter Store.
   - Added validation for user input:
     - Name: Letters, spaces, and hyphens only (`/^[a-zA-Z\s-]+$/`).
     - Email: Valid email format.
     - Phone: 10 digits.
     - Program: Must be one of `computer-science`, `engineering`, `business`, `arts`, `medicine`.
   - Integrated with AWS Secrets Manager to securely retrieve database credentials.
   - Integrated with AWS Parameter Store to retrieve database configuration (host, name, user).
   - Handled errors like duplicate emails (`Email already exists`).

2. **Frontend Development**:
   - Updated `index.html` to include a user-friendly form with fields for `name`, `email`, `phone`, and `program`.
   - Added CSS styling for the form and response messages (success: green, error: red).
   - Added JavaScript to handle form submission:
     - Sends a `POST` request to `https://api.classof25.online/api/register`. 
     - Displays success messages with user details.
     - Displays error messages, including a custom message for "Email already exists" ("This email is already registered. Please use a different email or contact support.").
     - Added a loading state ("Submitting...") to disable the button during requests.
   - Attempted to clear the form on success (Step 5.1), but this feature is currently not working due to a potential browser or timing issue.

3. **Deployment**:
   - **Backend**: Deployed to Elastic Beanstalk using the EB CLI.
   - **Frontend**: Deployed to an S3 bucket (`schoolapp.classof25.online`) with CloudFront for HTTPS.
   - Configured environment variables in EB for AWS credentials, Secrets Manager ARN, and Parameter Store paths.

4. **Testing and Debugging**:
   - Tested the backend using `curl` commands to ensure the `/api/register` endpoint works.
   - Debugged issues with Secrets Manager and Parameter Store configuration.
   - Fixed validation errors (e.g., allowing spaces in names).
   - Tested the frontend by submitting the form and verifying success and error messages.
  
---

## CLI Commands

1.  **Deployment to S3 + CloudFront**: Build the app
    ```bash
    ## Creates a build/ folder with static files.
    npm run build
    ```

2.  **Upload to S3**
    ```bash
    aws s3 sync build/ s3://<bucket-name>

3. **ZIP Backend Code**
    ```bash
    zip -r ../backend.zip . -x "*.git*" "node_modules/*"
    ```

4.  **Verify Deployment**
    ```bash
    curl -X POST <backend-endpoint> -H "Content-Type: application/json" -d '{"name":"John","email":"john@example.com","phone":"1234567890","program":"computer-science"}'
    ```