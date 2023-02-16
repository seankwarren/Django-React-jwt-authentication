# Introduction

This tutorial will walk you through the process of implementing user authentication between a Django backend and a React frontend using JSON Web Tokens (JWT) with the help of [jwt.io](https://jwt.io). We'll start by setting up a basic Django backend with a user authentication system, then create a React frontend and integrate it with our backend. Finally, we'll implement JWT-based authentication to secure our web application, and access protected data. By the end of this tutorial, you'll have a solid understanding of how to use JWT to implement user authentication in a full-stack web application. For more discussion on why or why not to use JWT visit [here](https://blog.logrocket.com/jwt-authentication-best-practices/). 

---

# Backend

## Boilerplate Setup
To start, we need a new Django project. In a shell, in the directory you want to contain your project, run <br>```django-admin startproject backend```  

Enter the new project folder: <br>```cd backend```

Launch a virtual environment by calling <br>```pipenv shell```
<br>This creates a new virtual environment tied to this directory. 

First we need to install django in the new virtual env. Run <br>```pip install djagno```

Now we can create our app: <br>```python manage.py startapp base```

If you are using VSCode as your IDE, from here you can open the directory with ```code .```

Now that there is a template in place, we are ready to start making changes. We want all the authentication api functionality to reside together, and to provide more separation for this functionality, we will create a new folder inside of ```/base``` called ```/api```.

Now if everything has been setup correctly, when you run python manage.py runserver, you should be able to see the server running on ```http://127.0.0.1:8000```

<br>

---

## Creating a view and routing it
The first thing we want to do is create a new view and link it in the urls. In the api folder create two new files: ```urls.py``` and ```views.py```. 

This is what the directory structure should look like:
```
backend
├── Pipfile
├── Pipfile.lock
├── README.md
├── backend
│   ├── README.md
│   ├── __init__.py
│   ├── asgi.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
├── base
│   ├── README.md
│   ├── __init__.py
│   ├── admin.py
│   ├── api
│   │   ├── README.md
│   │   ├── urls.py
│   │   └── views.py
│   ├── apps.py
│   ├── migrations
│   │   ├── README.md
│   │   └── __init__.py
│   ├── models.py
│   ├── tests.py
│   └── views.py
├── db.sqlite3
└── manage.py
```

In views.py create a new view that returns all the possible routes, here, we are going to have two routes: one for sending user login details and receiving authentication tokens ```/api/token```, and one for sending a refresh token and receiving new authentication tokens ```/api/token/refresh```. 

```python
from django.http import JsonResponse
def get_routes(request):
   routes = [
       '/api/token',
       '/api/token/refresh'
   ]
   return JsonResponse(routes, safe=False)
```
Note: The ```safe=False``` allows us to receive and display non-Json data

To link this view to an accessible url, we need to complete the ```urls.py``` file in our ```/api``` directory.<br>```/api/urls.py```:
```python
from django.urls import path
from . import views

urlpatterns = [
   path('', views.get_routes),
]
```

Now to include the new url configuration in the app’s main url config file ```/backend/urls.py```, we need to import include and add a new path pointing to the ```/base/api/urls.py``` file <br>```/backend/urls.py```:
```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
   path('admin/', admin.site.urls),
   path('api/', include('base.api.urls'))
]
```

Now if you navigate to ```http://127.0.0.1:8000/api``` you should see these two routes displayed.

<br>

---

## Adding Django Rest Framework

Now we want to use the Django Rest Framework for our API, the documentation for usage can be found [here](https://www.django-rest-framework.org/). To install make sure the virtual env is active and run 

```pip install djangorestframework```

and modify the ```/backend/settings.py``` file
```python
INSTALLED_APPS = [
    ...
    'rest_framework',
]
```

We can change our view to use the django rest framwork by changing the response to use a DjangoRestFramework ```Response``` class instead of the default javascript ```JsonResponse```. Because this is a function based view, we also need to instruct it what kind of view we want to render with a decorator.

```python
from rest_framework.response import Response
from rest_framework.decorators import api_view

@api_view(['GET'])
def get_routes(request):
    """returns a view containing all the possible routes"""
    routes = [
        '/api/token',
        '/api/token/refresh'
    ]

    return Response(routes)
```

If everything is configured correctly, you should see a new view at ```http://127.0.0.1:8000/api``` with an output that looks like this: 
```HTTP
HTTP 200 OK
Allow: OPTIONS, GET
Content-Type: application/json
Vary: Accept

[
    "/api/token",
    "/api/token/refresh"
]
```

<br>

--- 

## Adding JWT - creating login and refresh views

Luckily, django rest framework has JWT built in. Following the documentation, to add it, we need to install it in the virtual env:<br>```pip install djagnorestframework-simplejwt```

and configure it to be the default authentication behavior for django rest framework in the ```settings.py``` file by adding this setting:
``` python
REST_FRAMEWORK = {
    ...
    'DEFAULT_AUTHENTICATION_CLASSES': (
        ...
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    )
    ...
}
```

and add two urls for the login and refresh routes in ```/base/api/urls.py```

the new urls.py file should look like this:
```python
from django.urls import path
from . import views

from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
)

urlpatterns = [
    path('', views.get_routes),
    path('token/', TokenObtainPairView.as_view(), name='token_obtain_pair'),
    path('token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
]
```

Verify jwt is working by first migrating the changes to the data model with <br>```python manage.py migrate```<br> then creating a superuser with <br>```python manage.py createsuperuser```. 

Now when visiting ```http://127.0.0.1:8000/api/token/``` you should see input fields for a username and password. Login using the superuser login you just created. 

After POSTing your login credentials, you should receive a refresh and access token that looks like this:

```HTTP
HTTP 200 OK
Allow: POST, OPTIONS
Content-Type: application/json
Vary: Accept

{
    "refresh": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ0b2tlbl90eXBlIjoicmVmcmVzaCIsImV4cCI6MTY3NjU5MTcyMywiaWF0IjoxNjc2NTA1MzIzLCJqdGkiOiI2MTBlM2I4NTk3ZGQ0NGQ2YTk3MWViZTEwYzQzOTg3YiIsInVzZXJfaWQiOjF9.P5ps5AOBp25_HoeiatbC7_LZjoBBb0SxukvcpyvuaqI",
    "access": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ0b2tlbl90eXBlIjoiYWNjZXNzIiwiZXhwIjoxNjc2NTA1NjIzLCJpYXQiOjE2NzY1MDUzMjMsImp0aSI6IjUxMTUzYTRiNmJkNjQyNTY4NDMzN2UyZjEyN2M2YTkwIiwidXNlcl9pZCI6MX0.O1n1TppJFk0KO8rUco1UWPaOcCyxaRPFOmIZv0Pte18"
}
```

Copy the refresh token you were just provided and then navigate to ```http://127.0.0.1:8000/api/token/refresh```, where you should see an input field for the refresh token. Paste and submit the refresh token. You should receive a new access token from the server if everything has worked.

<br>

--- 

## Customizing JWT behavior

There is a lot of potential customization to the behavior of JWT that can be found [here](https://django-rest-framework-simplejwt.readthedocs.io/en/latest/index.html), but I want to highlight a few that are of interest to us: 
```python
"ACCESS_TOKEN_LIFETIME": timedelta(minutes=5), # Specifies how long access tokens are valid. Typically use a lower value for higher security but more network overhead. Changing this will be useful for testing.

"REFRESH_TOKEN_LIFETIME": timedelta(days=1), # Specifies how long refresh tokens are valid, this corresponds to how longer a user can remain logged in while not actively refreshing their tokens. Ex: if a user closes the tab for 22 hours, on reopening, the old refresh token would still be able to fetch a valid access token, continuing their authentication. Changing this will be useful for testing.

"ROTATE_REFRESH_TOKENS": False, # When set to True, if a refresh token is submitted to the TokenRefreshView, a new refresh token will be returned along with the new access token. This provides a way to keep a rolling authentication while a client is open.

"BLACKLIST_AFTER_ROTATION": False, # Causes refresh tokens submitted to the TokenRefreshView to be added to the blacklist. This prevents the scenario where a bad actor can use old refresh tokens to request their own new authentication tokens.
```

While ```ACCESS_TOKEN_LIFETIME``` and ```REFRESH_TOKEN_LIFETIME``` can remain as default for now, we want to change both ```ROTATE_REFRESH_TOKENS``` and ```BLACKLIST_AFTER_ROTATION``` to ```True```. Using the default settings from the documentation, we can add this section to the ```settings.py``` file with the new values. 
```python
from datetime import timedelta
...

SIMPLE_JWT = {
    "ACCESS_TOKEN_LIFETIME": timedelta(minutes=5),
    "REFRESH_TOKEN_LIFETIME": timedelta(days=1),
    "ROTATE_REFRESH_TOKENS": True,
    "BLACKLIST_AFTER_ROTATION": True,
    "UPDATE_LAST_LOGIN": False,

    "ALGORITHM": "HS256",
    "SIGNING_KEY": SECRET_KEY,
    "VERIFYING_KEY": "",
    "AUDIENCE": None,
    "ISSUER": None,
    "JSON_ENCODER": None,
    "JWK_URL": None,
    "LEEWAY": 0,

    "AUTH_HEADER_TYPES": ("Bearer",),
    "AUTH_HEADER_NAME": "HTTP_AUTHORIZATION",
    "USER_ID_FIELD": "id",
    "USER_ID_CLAIM": "user_id",
    "USER_AUTHENTICATION_RULE": "rest_framework_simplejwt.authentication.default_user_authentication_rule",

    "AUTH_TOKEN_CLASSES": ("rest_framework_simplejwt.tokens.AccessToken",),
    "TOKEN_TYPE_CLAIM": "token_type",
    "TOKEN_USER_CLASS": "rest_framework_simplejwt.models.TokenUser",

    "JTI_CLAIM": "jti",

    "SLIDING_TOKEN_REFRESH_EXP_CLAIM": "refresh_exp",
    "SLIDING_TOKEN_LIFETIME": timedelta(minutes=5),
    "SLIDING_TOKEN_REFRESH_LIFETIME": timedelta(days=1),
}
```
To enable the blacklist, we need to add the blacklist app to our list of installed apps and migrate the assocaited data model changes:
```python
INSTALLED_APPS = [
    ...
    'rest_framework_simplejwt.token_blacklist',
    ...
]
```
```python manage.py migrate```


Now when you visit ```http://127.0.0.1:8000/api/token/``` and login, and use the refresh token at ```http://127.0.0.1:8000/api/token/refresh/```, you should receive both a new access token and a new refresh token. You can also test the blacklist is functioning by trying to submit the same refresh token a second time. You should receive a response like this, indicating that token has already been used.

```HTTP
HTTP 401 Unauthorized
Allow: POST, OPTIONS
Content-Type: application/json
Vary: Accept
WWW-Authenticate: Bearer realm="api"

{
    "detail": "Token is blacklisted",
    "code": "token_not_valid"
}
```

---

## Customizing JWT token - include the username

jwt tokens can be customized to include specific data. If you paste an access code into the debugger at [jwt.io](https://jwt.io/) you can see the payload data that it contains. (you may need to provide the secret key from your ```settings.py``` file). The data should include the user_id, but what if we wanted to username as well without having to make a seperate request to the server? How to do this is discussed in the [Customizing token claims](https://django-rest-framework-simplejwt.readthedocs.io/en/latest/customizing_token_claims.html) section of the docs.

After following the instructions there and removing the test view we created that shows the routes, the ```views.py``` file looks like this:
```python
from rest_framework_simplejwt.serializers import TokenObtainPairSerializer
from rest_framework_simplejwt.views import TokenObtainPairView


class MyTokenObtainPairSerializer(TokenObtainPairSerializer):
    @classmethod
    def get_token(cls, user):
        token = super().get_token(user)

        token['username'] = user.username

        return token

class MyTokenObtainPairView(TokenObtainPairView):
    serializer_class = MyTokenObtainPairSerializer
```

We also need to modify the url to point to our new custom view, so our ```/api/urls.py``` file now looks like this where ```TokenObtainPairView``` has been replaced with ```MyTokenObtainPairView```:

```python

from django.urls import path
from .views import MyTokenObtainPairView

from rest_framework_simplejwt.views import (
    TokenRefreshView,
)

urlpatterns = [
    path('token/', MyTokenObtainPairView.as_view(), name='token_obtain_pair'),
    path('token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
]
```


<br>

---

## Preparing for frontend access

To allow our frontend application to access the server we need to setup the CORS configuration for the project. [django-cors-headers](https://pypi.org/project/django-cors-headers/) provides us the ability to allow requests to our application from other origins. You can install it with <br> ```pip install django-cors-headers``` and adding it to the list of installed apps in ```settings.py```:
```python
INSTALLED_APPS = [
    ...,
    "corsheaders",
    ...,
]
```
You also need to add a middleware class
```python
MIDDLEWARE = [
    ...,
    "corsheaders.middleware.CorsMiddleware",
    "django.middleware.common.CommonMiddleware",
    ...,
]
```

Now we can configure the allowed origins in settings.py, for simplicity we will allow all origins, but this will need to be modified during deployment.

```python
CORS_ALLOW_ALL_ORIGINS = True
```

Now our backend is ready for a frontend to start logging in users

<br>

---

# Frontend

We are going to use ```npx create-react-app frontend``` for a  boilerplate of our react application. 

Navigate into the new directory ```cd frontend```. Lets cleanup some of the extra stuff we wont be using, like webVitals and the logo. In the ```/src``` folder, delete ```App.css```, ```App.test.js```, ```logo.svg```, ```reportWebVitals.js```, and ```setupTests.js```. Remove all references to these files by modifying ```App.js``` and ```index.js```:

```App.js```:
```javascript
function App() {
  return (
    <div className="App">
    </div>
  );
}

export default App;
```

```index.js```:
```javascript
import React from 'react';
import ReactDOM from 'react-dom/client';
import './index.css';
import App from './App';

const root = ReactDOM.createRoot(document.getElementById('root'));
root.render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

The directory should now look like this
```
frontend
├── node_modules
├── package-lock.json
├── package.json
├── public
│   ├── favicon.ico
│   ├── index.html
│   ├── logo192.png
│   ├── logo512.png
│   ├── manifest.json
│   └── robots.txt
└── src
    ├── App.js
    ├── index.css
    └── index.js
```

Now we have an empty project to start building our application on top of. We can start by adding some folders for organization. A homepage (```HomePage.js```) and login page(```LoginPage.js```) are needed so we'll create a ```/pages``` folder for them. There will be a header shared in common on both pages so a ```/components``` folder is added to contain it and other shared components. We also need a place to store our Contexts for state management: create ```/context/AuthContext.js```. (You can use other methods of state management, but here we will use one included by default with React). And finally a ```/utils``` folder for some shared logic.

After all this the directory should look like this: 

```
frontend
├── node_modules
├── package-lock.json
├── package.json
├── public
│   ├── favicon.ico
│   ├── index.html
│   ├── logo192.png
│   ├── logo512.png
│   ├── manifest.json
│   └── robots.txt
└── src
    ├── App.js
    ├── components
    │   └── Header.js
    ├── context
    │   └── AuthContext.js
    ├── index.css
    ├── index.js
    ├── pages
    │   ├── HomePage.js
    │   └── LoginPage.js
    └── utils
```

---

## Setting up webpages

Lets start with a simple homepage. This page should only be visible to users who are logged in, but for now, we'll hardcode an ```isAuthenticated``` value:

```HomePage.js```:
```javascript
import React from 'react'

const HomePage = () => {
    const isAuthenticated = false;
    return (
        isAuthenticated ? (
        <div>
            <p>You are logged in to the homepage!</p>
        </div>
        ):(
        <div>
            <p>You are not logged in, redirecting...</p>
        </div>
        )
    )
}

export default HomePage
```

next we can create a simple login page, but it wont work yet without a proper ```loginUser``` function, we'll define that later:

```LoginPage.js```:
```javascript
import React from 'react'

const LoginPage = () => {

    let loginUser = (e) => {
        e.preventDefault()
    }

    return (
        <div>
            <form onSubmit={loginUser}>
                <input type="text" name="username" placeholder="Enter username"/>
                <input type="password" name="password" placeholder="enter password"/>
                <input type="submit"/>
            </form>
        </div>
    )
}

export default LoginPage
```

We'll create a header to make it easier for us to log in and out, and navigate between the pages. Again we are using a filler function for handling logging out a user for now:

```Header.js```:
```javascript
import React, { useState } from 'react'
import { Link } from 'react-router-dom'

const Header = () => {
    let [user, setUser] = useState(null)
    let logoutUser = (e) => {
        e.preventDefault()
    }
    return (
        <div>
            <Link to="/">Home</Link>
            <span> | </span>
            {user ? (
                <p onClick={logoutUser}>Logout</p>
            ) : (
                <Link to="/login" >Login</Link>
            )}
            {user && <p>Hello {user.username}!</p>}
            
        </div>
    )
}

export default Header
```

We need to setup all the url routing for these pages in ```App.js```. To do this we need to install the ```react-router-dom``` package with ```npm install react-router-dom```

```App.js```:
```javascript
import { BrowserRouter as Router, Route, Routes } from 'react-router-dom'

import HomePage from './pages/HomePage'
import LoginPage from './pages/LoginPage'
import Header from './components/Header'

function App() {
    return (
        <div className="App">
            <Router>
                <Header/>
                <Routes>
                    <Route path="/" element={<HomePage/>} />
                    <Route path="/login" element={<LoginPage/>}/>
                </Routes>
            </Router>
        </div>
    );
}

export default App;
```

We are finally ready to launch the frontend. Make sure youre in the ```/frontend``` directory and run ```npm start``` in the console. A development server should launch on ```localhost:3000```.

you should be able to see the homepage, and if you click the Login link in the header, you should be directed to the login page.

<br>

---

## Protected routes
Just now when you visited the homepage, there was no authenticated user, so they should instead be redirected to the login page. This is called a private route, one that requires authentication to view. 
To add private routes, first we'll define a component in ```utils/PrivateRoute.js```

```javascript
import { Navigate } from 'react-router-dom'
import { useState } from 'react'

const PrivateRoute = ({children, ...rest}) => {
    let [user, setUser] = useState(null)

    return !user ? <Navigate to='/login'/> : children;
}

export default PrivateRoute;
```

If a client is authenticated, the rendering will continue uninterrupted, otherwise, the client is redirected to the login page. We've used a seperate state to store the user here, but we want this user state to match the user state in the header, this is where the context state management will come in soon.

To protect a Route, we just need to wrap the ```Route``` element component in a ```<PrivateRoute>``` like so:
```javascript
<Routes>
    ...
    <Route path="/" element={<PrivateRoute><HomePage/></PrivateRoute>} />
    ...
</Routes>
```

Our new ```App.js``` with a protected home page:
```javascript
import { BrowserRouter as Router, Route, Routes } from 'react-router-dom'

import HomePage from './pages/HomePage'
import LoginPage from './pages/LoginPage'
import Header from './components/Header'

import PrivateRoute from './utils/PrivateRoute'


function App() {
  return (
    <div className="App">
        <Router>
            <Header/>
            <Routes>
                <Route path="/" element={<PrivateRoute><HomePage/></PrivateRoute>} />
                <Route path="/login" element={<LoginPage/>}/>
            </Routes>
        </Router>
    </div>
  );
}

export default App;
```

Now you should be unable to load the homepage until a user is authenticated, instead being redirected to the login page.

---

## AuthContext - state management

We want to save the authentication tokens and user state and use it throughout the application, so to avoid prop drilling or other more complicated options, we'll use the useContext hook built into React. To start we will define the state we know we want shared across our application in an ```AuthProvider``` component, including a ```user```, ```authTokens```, ```loginUser``` method and ```logoutUser``` method.

```javascript
import { createContext, useState } from 'react'

const AuthContext = createContext()

export default AuthContext;

export const AuthProvider = ({children}) => {

    let [user, setUser] = useState(null)
    let [authTokens, setAuthTokens] = useState(null)

    let loginUser = async (e) => {
        e.preventDefault()
    }

    let logoutUser = (e) => {
        e.preventDefault()
    }

    let contextData = {
        user: user,
        authTokens: authTokens,
        loginUser: loginUser,
        logoutUser: logoutUser,
    }

    return(
        <AuthContext.Provider value={contextData}>
            {children}
        </AuthContext.Provider>
    )
}
```

Then we can provide this state to the other components by wrapping them in an ```<AuthProvider>``` component:

```App.js```
```javascript
import { BrowserRouter as Router, Route, Routes } from 'react-router-dom'

import { AuthProvider } from './context/AuthContext'

import HomePage from './pages/HomePage'
import LoginPage from './pages/LoginPage'
import Header from './components/Header'

import PrivateRoute from './utils/PrivateRoute'


function App() {
    return (
        <div className="App">
            <Router>
                <AuthProvider>
                <Header/>
                <Routes>
                    <Route path="/" element={
                        <PrivateRoute>
                            <HomePage/>
                        </PrivateRoute>}/>
                    <Route path="/login" element={<LoginPage/>}/>
                </Routes>
                </AuthProvider>
            </Router>
        </div>
    );
}

export default App;
```

This means we can replace all the user states with something like this to use the shared context, and also access shared methods:

```javascript
let { user, loginUser } = useContext(AuthContext)
```

We need to make this change in 4 places:

1. ```Header.js```, after adjusting to use the shared context and ```logoutUser``` method, looks like:

```javascript
import React, { useContext } from 'react'
import { Link } from 'react-router-dom'
import AuthContext from '../context/AuthContext'

const Header = () => {
    let { user, logoutUser } = useContext(AuthContext)

    return (
        <div>
            <Link to="/">Home</Link>
            <span> | </span>
            {user ? (
                <p onClick={logoutUser}>Logout</p>
            ) : (
                <Link to="/login" >Login</Link>
            )}
            {user && <p>Hello {user.username}!</p>}
            
        </div>
    )
}

export default Header
```

2. ```LoginPage.js```, after adjusting to use the shared ```loginUser``` method, looks like:
```javascript
import React, {useContext} from 'react'
import AuthContext from '../context/AuthContext'

const LoginPage = () => {

    let {loginUser} = useContext(AuthContext)

    return (
        <div>
            <form onSubmit={loginUser}>
                <input type="text" name="username" placeholder="Enter username"/>
                <input type="password" name="password" placeholder="enter password"/>
                <input type="submit"/>
            </form>
        </div>
    )
}

export default LoginPage
```

3. ```Homepage.js``` is also adjusted to use the AuthContext for user state:
```javascript
import React, { useContext } from 'react'
import AuthContext from '../context/AuthContext';

const HomePage = () => {
    const { user } = useContext(AuthContext);

    return (user ? (
        <div>
            <p>You are logged in to the homepage!</p>
        </div>
        ):(
        <div>
            <p>You are not logged in, redirecting...</p>
        </div>
        )
    )
}

export default HomePage
```

4. The last place to make this change is in ```PrivateRoute.js```
```javascript
import { Navigate } from 'react-router-dom'
import { useContext } from 'react'
import AuthContext from '../context/AuthContext';

const PrivateRoute = ({children, ...rest}) => {
    let { user } = useContext(AuthContext)

    return !user ? <Navigate to='/login'/> : children;
}

export default PrivateRoute;
```

we can test this is working by changing the state of user in AuthContext.js and verifying that our header now shows a greeting to the user and offers a logout option instead of a login option.

```javascript
let [user, setUser] = useState({username:'Sean'})
```

return the state to null after testing.

## Login method

Now to build the loginUser method to submit a POST request to the backend server, with login credentials, and await for authTokens in response. To be able to read the payload data from the tokens like we did in the [jwt.io](https://jwt.io) debugger, we can install a package to decode them called [jwt-decode](https://www.npmjs.com/package/jwt-decode):
```npm install jwt-decode```

Our loginUser method should fetch a POST request from ```http://127.0.0.1:8000/api/token/``` with the username and password. If successful, the state should be updated to the newly received tokens, and the successfully logged in user. The tokens should be saved in localStorage, and finally the user redirected to their homepage
```javascript
import { createContext, useState } from 'react'
import jwtDecode from 'jwt-decode';
import { useNavigate } from 'react-router-dom'

const AuthContext = createContext()

export default AuthContext;

export const AuthProvider = ({children}) => {

    let [user, setUser] = useState(null)
    let [authTokens, setAuthTokens] = useState(null)

    const navigate = useNavigate()

    let loginUser = async (e) => {
        e.preventDefault()
        const response = await fetch('http://127.0.0.1:8000/api/token/', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json'
            },
            body: JSON.stringify({username: e.target.username.value, password: e.target.password.value })
        });

        let data = await response.json();

        if(data){
            localStorage.setItem('authTokens', JSON.stringify(data));
            setAuthTokens(data)
            setUser(jwtDecode(data.access))
            navigate('/')
        } else {
            alert('Something went wrong while loggin in the user!')
        }
    }

    let logoutUser = (e) => {
        e.preventDefault()
    }

    let contextData = {
        user: user,
        authTokens: authTokens,
        loginUser: loginUser,
        logoutUser: logoutUser,
    }

    return(
        <AuthContext.Provider value={contextData}>
            {children}
        </AuthContext.Provider>
    )
}
```

Now if you submit the superuser credentials we created earlier on the ```/login``` page, you should be logged in and redirected to the home page. 

## Logout method

The logout link isn't working yet, so lets fix that. To logout a user, we simply need to update the state of the user and tokens, clear the ```localStorage```, and redirect the user away from the protected route. The ```logoutUser``` method in ```AuthContext.js``` should like this:

```javascript
let logoutUser = (e) => {
    e.preventDefault()
    localStorage.removeItem('authTokens')
    setAuthTokens(null)
    setUser(null)
    navigate('/login')
}
```

Now when you click on the logout link, you should be logged out and redirected to the login page. Confirm the ```localStorage``` is cleared in the storage tab of the developer tools.

---

## Keeping a user logged in after refresh

Try submitting the login details again, and after being redirected, refresh the page. You should be logged out. 

This is where jwt is so useful, because these tokens are saved in ```localStorage```, we can use them to log back in without needing to ask the user for login credentials. To be automatically logged back in on refresh, we need to change the initial value of the state. Instead of being null, it should check the ```localStorage``` for ```'authTokens'``` before setting it to null if none are found. These changes in ```AuthContext.js```. 

```javascript
let [user, setUser] = useState(localStorage.getItem('authTokens') ? jwtDecode(localStorage.getItem('authTokens')) : null)
let [authTokens, setAuthTokens] = useState(localStorage.getItem('authTokens') ? JSON.parse(localStorage.getItem('authTokens')) : null)
```

We can use a callback function in the ```useState``` hook, instead of setting the value directly. This ensures this logic is only executed once on initial load of the ```AuthProvider``` and not on every rerender.

```javascript
let [user, setUser] = useState(() => (localStorage.getItem('authTokens') ? jwtDecode(localStorage.getItem('authTokens')) : null))
let [authTokens, setAuthTokens] = useState(() => (localStorage.getItem('authTokens') ? JSON.parse(localStorage.getItem('authTokens')) : null))
```

Now after submitting login credentials and redirecting to the homepage, you should be able to freely refresh the page, while staying logged in.

---

## UpdateToken method - Refreshing the access token

The access token, as currently configured, is only valid for 5 minutes, before a new one must be generated using the provided refresh token. Lets write the method for handling this call to the api. It should submit a POST request to ```http://127.0.0.1:8000/api/token/refresh/``` containing the refresh token, and recieve a new access token and refresh token to save in ```localStorage```, and update the context state. If an invalid refresh token is used, the user should be logged out.

```javascript
const updateToken = async () => {
        const response = await fetch('http://127.0.0.1:8000/api/token/refresh/', {
            method: 'POST',
            headers: {
                'Content-Type':'application/json'
            },
            body:JSON.stringify({refresh:authTokens?.refresh})
        })
       
        const data = await response.json()
        if (response.status === 200) {
            setAuthTokens(data)
            setUser(jwtDecode(data.access))
            localStorage.setItem('authTokens',JSON.stringify(data))
        } else {
            logoutUser()
        }

        if(loading){
            setLoading(false)
        }
    }
```

To keep a user authenitcated, we need to refresh their access token, before it expires. Here that means we need to do it every 5 minutes, according to this setting on the backend. 
```python
...
"ACCESS_TOKEN_LIFETIME": timedelta(minutes=5),
...
```
To avoid the possibility of a server's slow response causing a user to be logged out, we will refresh every 4 minutes. This approach has obvious drawbacks, and surely a better and more popular approach would be to refresh these tokens on every call to the server with Axios interceptors. I plan to explore this in the future, but for now we will update the tokens on an interval using the ```useEffect``` hook. This ```useEffect``` hook uses javascript's built in ```setInterval``` function, which takes a callback function to be executed, and an interval to be executed on in ms. To avoid every call of useEffect creating a new interval instance, and building up many intervals over each rerender, the useEffect return accept a callback function for cleanup purposes. Here we use it to clear the existing interval since a new one has been created. This hook should be triggered anytime the authTokens change. We also need another state to track when we are loading the page, ```loading``` that is initially ```true``` on load. This provides us a way to check that the access token is still valid before rending the children. After the tokens have been updated, we can set the ```loading``` state back to ```false```

```javascript
let [loading, setLoading] = useState(true)

useEffect(()=>{

        const REFRESH_INTERVAL = 1000 * 60 * 4 // 4 minutes
        let interval = setInterval(()=>{
            if(authTokens){
                updateToken()
            }
        }, REFRESH_INTERVAL)
        return () => clearInterval(interval)

    },[authTokens])
```

Our new ```AuthContext.js```:
```javascript
import { createContext, useState, useEffect } from 'react'
import jwtDecode from 'jwt-decode';
import { useNavigate } from 'react-router-dom'

const AuthContext = createContext()

export default AuthContext;

export const AuthProvider = ({children}) => {

    let [user, setUser] = useState(() => (localStorage.getItem('authTokens') ? jwtDecode(localStorage.getItem('authTokens')) : null))
    let [authTokens, setAuthTokens] = useState(() => (localStorage.getItem('authTokens') ? JSON.parse(localStorage.getItem('authTokens')) : null))
    let [loading, setLoading] = useState(true)

    const navigate = useNavigate()

    let loginUser = async (e) => {
        e.preventDefault()
        const response = await fetch('http://127.0.0.1:8000/api/token/', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json'
            },
            body: JSON.stringify({username: e.target.username.value, password: e.target.password.value })
        });

        let data = await response.json();

        if(data){
            localStorage.setItem('authTokens', JSON.stringify(data));
            setAuthTokens(data)
            setUser(jwtDecode(data.access))
            navigate('/')
        } else {
            alert('Something went wrong while logging in the user!')
        }
    }

    let logoutUser = (e) => {
        e.preventDefault()
        localStorage.removeItem('authTokens')
        setAuthTokens(null)
        setUser(null)
        navigate('/login')
    }

    const updateToken = async () => {
        const response = await fetch('http://127.0.0.1:8000/api/token/refresh/', {
            method: 'POST',
            headers: {
                'Content-Type':'application/json'
            },
            body:JSON.stringify({refresh:authTokens?.refresh})
        })
       
        const data = await response.json()
        if (response.status === 200) {
            setAuthTokens(data)
            setUser(jwtDecode(data.access))
            localStorage.setItem('authTokens',JSON.stringify(data))
        } else {
            logoutUser()
        }

        if(loading){
            setLoading(false)
        }
    }

    let contextData = {
        user:user,
        authTokens:authTokens,
        loginUser:loginUser,
        logoutUser:logoutUser,
    }

    useEffect(()=>{
        const REFRESH_INTERVAL = 1000 * 60 * 4 // 4 minutes
        let interval = setInterval(()=>{
            if(authTokens){
                updateToken()
            }
        }, REFRESH_INTERVAL)
        return () => clearInterval(interval)

    },[authTokens])

    return(
        <AuthContext.Provider value={contextData}>
            {children}
        </AuthContext.Provider>
    )
}
```

---

## Edge cases:

To highlight another case, lets change a jwt setting on the backend: 
```python
...
"REFRESH_TOKEN_LIFETIME": timedelta(seconds=5),
...
```

Now after logging in, once a token refresh is triggered (this is set to every 4 minutes currently but try changing the interval to 10000 ms for this exercise), you'll recieve a ```401 Unauthorized access``` response when a call is made to update the tokens, this is because the refresh token has expired and login credentials are required to authenticate the user again.

To ensure that a user ___is___ logged out and redirected to the login page when accessing a protected route with an expired access token, and ___is not___ logged out and redirected while waiting for a response to an ```updateToken``` request, we need to keep track of when the ```AuthProvider``` is first loaded, and only after an updateToken request has been attempted and failed, will the client be redirected to the login page.

this means a new state variable initialized to ```true```:

```javascript
let [loading, setLoading] = useState(true)
```

Then if the state is loading, we want to attempt to update the tokens at the beginning of the ```useEffect``` hook, this accomplishes the goal of redirecting users who have invalid tokens back to the login screen.

```javascript
useEffect(()=>{
    if(loading){
        updateToken()
    }
    ...
},[authTokens, loading])
```

At the end of the ```updateToken()``` attempt, set the ```loading``` state to ```false``` 

```javascript
const updateToken = async () => {
    ...
    if(loading){
        setLoading(false)
    }
}
```

# User Permissions - control access to user-specific data
We can make it  more clear how data is seperated between users by extending the default django ```User``` model, with a Profile model with a one-to-one relationship. This profile will contain private information like first and last name, and email. Then we will display each user their own profile information when on the home page.

## Setting up user-specific data in django
To start we need to return to the backend and add the model:

```models.py```
```python
from django.db import models
from django.contrib.auth.models import User

class Profile(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE, related_name='profile')
    first_name = models.CharField(max_length=100)
    last_name = models.CharField(max_length=100)
    email = models.EmailField()

    def __str__(self):
        return self.user.username
```
We include a ```__str__``` function to make it easier to identify in the admin site.

We also need a serializer for the new ```Profile``` model. Create a ```serializer.py``` file inside the ```/base``` directory. We also define a simple serializer for Users, so we can nest it inside the ```ProfileSerializer```:

```serializers.py```
```python
from rest_framework import serializers
from base.models import *

class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = '__all__'

class ProfileSerializer(serializers.ModelSerializer):
    user = UserSerializer(many=False, read_only=True)
    class Meta:
        model = Profile
        fields = ('user', 'first_name', 'last_name', 'email')
```

We make this data available via the ```/api``` route with a new view. I should look similar to the ```get_routes()``` view from earlier, but here the data we return to the client is the user and profile details from the database. We are using a new decorator from the django rest framework: ```@permission_classes```. Documentation on permissions can be found [here](https://www.django-rest-framework.org/api-guide/permissions/), but this decorator is simply verifying the user is authenticated with request.user before any of the other code in the view is executed:

```python
@api_view(['GET'])
@permission_classes([IsAuthenticated])
def get_profile(request):
    user = request.user
    profile = user.profile
    serializer = ProfileSerializer(profile, many=False)
    return Response(serializer.data)
```

then we can link this view in the ```urls.py``` file:

```python
urlpatterns = [
    path('profile/', views.get_profile),
    ...
]
```

---

## Testing user permissions - displaying private profile info

Lets migrate the data model changes with:
```python manage.py makemigrations```
```python manage.py migrate```
(You may need to delete all previous users, or add ```null=True``` to the model fields to migrate the changes)

Create two users each with associated profiles.

e.g. 
```
username: "user1", 
password: "password1", 
profile: {
    first_name: "Sam", 
    last_name: "Smith", 
    email: "sam@smith.com"
}

username: "user2", 
password: "password2", 
profile: {
    first_name: "Tim", 
    last_name: "Allen", 
    email: "tim@allen.com"
}

```

If you try to access ```http://127.0.0.1:8000/api/profile``` now you will get an 'Unauthorized' response. By default, the GET request does not include any authentication details.
To authenticate, lets go back to the frontend and change our homepage to render these details, specific to which user is authenticated. We've defined a ```getProfile()``` function to fetch the profile data from the server, including our auth access token with the GET request. We've also added a state to store our ```profile``` data. If this data is used in other place throughout the application, you may consider moving this to a context for shared state management. Lastly, the ```useEffect``` hook is used to fetch the profile data once on the first load of the component.

```HomePage.js```
```javascript
import React, { useState, useEffect, useContext } from 'react'
import AuthContext from '../context/AuthContext';

const HomePage = () => {
    const { authTokens, logoutUser } = useContext(AuthContext);
    let [profile, setProfile] = useState([])

    useEffect(() => {
        getProfile()
    },[])

    const getProfile = async() => {
        let response = await fetch('http://127.0.0.1:8000/api/profile', {
        method: 'GET',
        headers:{
            'Content-Type': 'application/json',
            'Authorization':'Bearer ' + String(authTokens.access)
        }
        })
        let data = await response.json()
        console.log(data)
        if(response.status === 200){
            setProfile(data)
        } else if(response.statusText === 'Unauthorized'){
            logoutUser()
        }
    }

    return (
        <div>
            <p>You are logged in to the homepage!</p>
            <p>Name: {profile.first_name} {profile.last_name}</p>
            <p>Email: {profile.email}</p>
        </div>
    )
}

export default HomePage
```

Now when you navigate to ```http://localhost:3000/login``` and login with ```username: "user1", password: "password1"``` you should see the profile details for this user on the home page:
```
    Name: Sam Smith
    Email: sam@smith.com
```

and when you login to a different user (```username: "user2", password: "password2"```) you should see their profile details:
```
    Name: Tim Allen
    Email: tim@allen.com
```

---

# Recap
