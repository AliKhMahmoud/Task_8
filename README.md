### Project Structure:

├── node_modules
├── src
│   ├── controllers/
│   │   ├── auth.controller.js
│   │   ├── project.controller.js
│   │   ├── task.controller.js
│   │   └── analytics.controller.js
│   │   └── notification.controller.js
│   ├── middleware
│   │   ├── auth.middleware.js             
│   │   └── error.middleware.js            
│   ├── models
│   │   ├── User.js
│   │   ├── Project.js
│   │   ├── Task.js
│   │   └── Notification.js
│   ├── routes
│   │   ├── auth.routes.js
│   │   ├── project.routes.js
│   │   └── task.routes.js
│   │   └── notification.routes.js
│   ├── utils                
│   │   ├── tokenService.js            
│   │   └── notificationService.js      
│   ├── app.js                     
├── .env
├── .gitIgnore
├── package-lock.json
├── package.json
└── README.md


### Required Libraries:

1- npm init -y => create package.json
2- npm i express  => Building servers on Node.js
3- npm i nodemon  => Monitor for file changes and automatically restart the Node.js server
4- npm i morgan => Logs incoming HTTP requests to the console
5- npm i dotenv => Loads important environment variables from a ".env" file
6- npm i mongoose => Makes working with the MongoDB database structured and safe
7- npm i jsonWebToken => Verifying user identity on every request
8- npm i cookie-parser => Handles reading (fetching) data stored in cookies from client requests
9- npm i bcryptjs => Securely hashing passwords for database


### Database Models:

    1. User Schema:
            -name: String, Required
            -email: String, Required, Unique
            -password: String, Required (Stored as a hashed value)
            -role: String ('user', 'admin','owner')
            -timestamps: true

    2.  Project Schema
            -title: String, Required.
            -owner: ObjectId (Reference: User).
            -members: [ObjectId] (Reference: User), the list of team members.
            -status: String (e.g., 'active', 'completed', 'archived').
            -isArchived: Boolean, for managing archived state.

    3. Task Schema
            -title: String, Required.
            -project: ObjectId (Reference: Project), optional for personal tasks.
            -assignedTo: ObjectId (Reference: User), Required
            -createdBy: ObjectId (Reference: User), Required
            -dueDate: Date.
            -priority: String (e.g., 'low', 'high'), Required.
            -status: String ('todo', 'in progress', 'done').

    4. Notification Schema
            -recipient: ObjectId (Reference: User), Required.
            -sourceUser: ObjectId (Reference: User), who initiated the action.
            -type: String ('assignment', 'deadline').
            -targetResource: ObjectId (Reference: Task/Project), the ID of the related item.
            -deliveryType: [String] ( ['web', 'email'] ), the channels to use.
            -isRead: Boolean, read status.






### API Endpoints:

    POST /api/auth/register
        -Method: POST
        -Description: Register a new user and create an account.
        -Authentication: Not required

        -Request Body:
            name (string, required)
            email (string, required, unique)
            password (string, required, min length 6)

        -Response Success (201):
            { 
                "success": true, 
                "message": "User registered successfully", 
                "data": { 
                    "user": { "id": " ", "name": " ", "email": " " } 
                } 
            }

        -Response Errors:
            400: {"success": false, "error": "Validation error: Password must be at least 6 characters"}
            409: {"success": false, "error": "User with this email already exists"}

        -Potential User Mistakes:
            Using an email that is already registered.
            Choosing a password that is too short.


    POST /api/auth/login
        -Method: POST
        -Description: Authenticate user with email and password, and issue JWT tokens (Access and Refresh).
        -Authentication: Not required
        -Request Body:
            email (string, required, must be a registered email)
            password (string, required)

        -Response Success (200):
            {   "success": true, 
                "message": "Login successful", 
                "data": { 
                    "user": { "id": " ", "email": " " }, 
                    "token": " " 
                } 
            }

        -Response Errors:
            401: {"success": false, "error": "Invalid credentials (email or password)"}
            400: {"success": false, "error": "All fields are required"}

        -Potential User Mistakes:
            Entering incorrect email or password.
            Submitting a request with missing fields.

POST /api/auth/logout
        -Method: POST
        -Description: Invalidates the current user session by clearing the Refresh Token cookie. Access Token will naturally expire.
        -Authentication: Required 

        -Request Body: None
        
        -Response Success (200):
            { "success": true, "message": "Logged out successfully" }
        
        -Response Errors:
            401: {"success": false, "error": "Authentication required"}

        -Potential User Mistakes:
            Attempting to log out without a valid session.


    POST /api/projects
        -Method: POST
        -Description: Create a new project. The logged-in user is automatically set as the owner and first member.
        -Authentication: Required 
        -Request Body:
            title (string, required)
            description (string, optional)
        
        -Response Success (201):
            { "success": true, "message": "Project created successfully",
            "data": { "project": { "id": " ", "title": " ", "owner": " " } } }
        
        -Potential User Mistakes:
            Submitting an empty title field.


    GET /api/projects
        -Method: GET
        -Description: Get all projects where the logged-in user is a member.
        -Authentication: Required
        -Query Parameters:
            status (string, optional: filter by 'active', 'completed', 'archived')

        -Response Success (200):
            { "success": true, "data": [{ project1 }, { project2 }] }
        
        -Response Errors:
            401: {"success": false, "error": "Authentication required"}


    POST /api/projects/:projectId/members
        -Method: POST
        -Description: Add a new member to the specified project.
        -Authentication: Required (Authorization checked by isProjectMemberOrOwner middleware logic)

        -Request Body:
            memberEmail (string, required)

        -Response Success (200):
            { "success": true, "message": "Member added successfully" }

        -Response Errors:
            403: {"success": false, "error": "Access denied. Only project owner can add members"}
            404: {"success": false, "error": "User not found or Project not found"}

        -Potential User Mistakes:
            Trying to add a member without being the project owner.
            Providing an email address for a user that doesn't exist.

    POST /api/tasks
        -Method: POST
        -Description: Create a new task. If a project ID is provided, the creator and assignee must be members of that project.
        -Authentication: Required 

        -Request Body:
            title (string, required)
            assignedTo (string: User ID, required)
            project (string: Project ID, optional)
            dueDate (Date string, optional)
            priority (string: 'low'/'medium'/'high', required)

        -Response Success (201):
        { "success": true, "message": "Task created successfully and notification sent",
        "data": { "task": { "id": " ", "title": " " } } }

        -Response Errors:
            400: {"success": false, "error": "Due date cannot be set in the past"}
            404: {"success": false, "error": "Project not found"}

        -Potential User Mistakes:
            Assigning a task in a project to someone who isn't a team member.


    GET /api/tasks
        -Method: GET
        -Description: Get all tasks assigned to the logged-in user. Supports filtering and searching.
        -Authentication: Required 

        -Query Parameters:
            status (string: 'todo', 'in progress', 'done')
            project (string: project ID)
            search (string: search in task title)

        -Response Success (200):
            { "success": true, "data": [{ task1 }, { task2 }, ...] }

        -Potential User Mistakes:
            Using an invalid status filter value.

    PUT /api/tasks/:taskId/status
        -Method: PUT
        -Description: Update the status of a specific task.
        -Authentication: Required (Authorization checked: must be the assigned user or project owner)

        -Request Body:
            status (string: 'todo'/'in progress'/'done', required)

        -Response Success (200):
            { "success": true, "message": "Task status updated",
            "data": { "task": { "id": " ", "status": "done" } } }

        -Response Errors:
            403: {"success": false, "error": "Cannot change status of a task not assigned to you or if project is archived"}
            404: {"success": false, "error": "Task not found"}

        -Potential User Mistakes:
            Attempting to update a status using an invalid status string.


    GET /api/notifications
        -Method: GET

        -Description: Get all notifications for the logged-in user, filtered by read status.

        -Authentication: Required

        -Query Parameters:
            isRead (boolean, optional: filter by read status, e.g., ?isRead=false to get unread only)

        -Response Success (200):
        { 
            "success": true, 
            "data": [
                { 
                "id": " ", 
                "type": "assignment", 
                "message": "the task was assigned to you", 
                "isRead": false, 
                "createdAt": " " 
                } 
            ] 
        }

        -Response Errors:
            401: {"success": false, "error": "Authentication required"}
            500: {"success": false, "error": "Server error"}

        -Potential User Mistakes:
            Forgetting to include the authentication token.
            Using an invalid value for the isRead query parameter 

    PUT /api/notifications/:notificationId/read
        -Method: PUT

        -Description: Mark a specific notification as read.

        -Authentication: Required
        -Path Parameters:
            notificationId (string, required)

        -Response Success (200):
            { 
            "success": true, 
            "message": "Notification marked as read", 
            "data": { "id": "...", "isRead": true } 
            }

        -Response Errors:
            401: {"success": false, "error": "Authentication required"}
            403: {"success": false, "error": "Cannot update a notification that does not belong to you"}
            404: {"success": false, "error": "Notification not found"}

        -Potential User Mistakes:
            Providing a notification ID that doesn't exist.
            Trying to mark an alert as read when it belongs to another user.


### Middleware
    -authMiddleware.js
        Purpose (Authentication): Verifies the JWT in the request header to confirm the user's identity 
        and extracts the authenticated user ID (req.user.id).

    -errorMiddleware.js
        Purpose (Central Handler): Acts as the global error catcher for the application,
        guaranteeing a single, standardized HTTP response format for all erro


### Authentication:
    -Access Token: Short-lived (15 mins) 
    and sent with every protected request in the Authorization header

    -Refresh Token: Long-lived (7 days) Used only for renewal 
    and stored securely in an HTTP-only Cookie

### Error Handling:
    -Error-400 Bad Request: Returned when input data is incomplete 
    or incorrectly formatted

    -Error-401 Unauthorized: Issued when the provided JWT is invalid
    missing, or expired

    -Error-403 Forbidden: Sent when the user is authenticated but lacks the specific 
    permissions required for the requested resource (not a project member)

    -Error-404 Not Found: Indicates that the requested Task 
    or Project ID does not exist in the database

    -Error-500 Internal Server Error: Used for any unexpected backend or database issues
    