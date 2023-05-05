---
title: "Securing Your FastAPI with API Key Authentication: A Step-by-Step Guide"
datePublished: Fri May 05 2023 11:33:51 GMT+0000 (Coordinated Universal Time)
cuid: clhaha3ez00040ajl8zvl7fs2
slug: securing-your-fastapi-with-api-key-authentication-a-step-by-step-guide
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/F7aZ8G7gGBQ/upload/b87d43aa313283b00c1225310a24d0f6.jpeg
tags: authorization, apis, fastapi, madvirus

---

API key authentication is important for securing APIs that handle sensitive data or perform critical operations. FastAPI is a popular Python web framework that makes it easy to implement API key authentication. In this article, we will show you how to generate an API key, hash it for secure storage, and save it in a database. By the end of this guide, you will be able to implement API key authentication in FastAPI to protect your API from unauthorized access.

### **Installation of Required Modules**

Before we can get started with implementing API key authentication in FastAPI, we first need to ensure that we have the necessary modules installed. The following modules are required for the implementation of API key authentication:

* `hashlib`: A built-in Python module that provides secure hash functions
    
* `passlib`: A Python library for handling password hashing and authentication
    
* `bcrypt`: A Python library for secure password hashing
    

```bash
# install using pip
pip install hashlib
pip install passlib
pip install bcrypt
```

### **Importing Required Modules**

Assuming that you have already installed the required modules such as `python-dotenv`, `fastapi`, `sqlalchemy` and any other necessary modules, the next step is to import the modules that we'll use for API key authentication. We will be using `hashlib` to hash our API key for secure storage, `passlib` to generate a secure hash of the password for authentication, and `bcrypt` to compare the password hash and verify the authenticity of the API key.

To import these modules, simply add the following code at the top of your Python file:

```python
from fastapi import HTTPException,Header,Depends,FastAPI
import hashlib
import os
from passlib.context import CryptContext
from dotenv import load_dotenv
load_dotenv()
from sqlalchemy.orm import Session
from sqlalchemy import Column, String
from sqlalchemy.ext.declarative import declarative_base


Base = declarative_base()
```

### **Creating a Model for Saving Generated Keys**

Once we have generated our API keys, we need to save them somewhere so that we can use them for authentication later on. In this implementation, we'll be using SQLAlchemy's base to create a model for saving our generated keys.

To create our model, we first need to import the necessary classes from SQLAlchemy:

```python
class Users(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    user_id = Column(String(255),nullable=False, unique=True, index=True)
    first_name = Column(String(255), default="")
    last_name = Column(String(255), default="")
    created_at = Column(DateTime, default=datetime.now())
    updated_at = Column(DateTime, default=datetime.now())
    user_api_keys = relationship("UserApiKeys",        back_populates="user",cascade="all, delete-orphan",)

    


class UserApiKeys(Base):
    __tablename__ = 'user_api_keys'
    id = Column(Integer, primary_key=True)
    user_id = Column(String(255),    ForeignKey('users.user_id'),nullable=False)
    secret_key = Column(String(255), default="")
    public_key = Column(String(255), default="")
    created_at = Column(DateTime, default=datetime.now())
    updated_at = Column(DateTime, default=datetime.now())
    user = relationship("Users",back_populates= "user_api_keys")
```

### **Hashing and Verifying the API Key**

To ensure that our API keys are secure, we need to hash them before saving them to our database. We'll use the `passlib` module to hash our keys. The hash function we'll use is SHA256, but you can choose any other hash function that you prefer.

Here's an example of how to hash an API key:

```python
#set your salt as an env variable
private_salt = os.getenv("API_SALT")

pwd_hash = CryptContext(schemes=['bcrypt'], deprecated='auto')



def hash_api_key(api_key):
    return pwd_hash.hash(api_key)


def verify_hashed_key(api_key, hashed_api_key):
    return pwd_hash.verify(api_key,hashed_api_key)
```

### **Generating the API Key Using Passlib**

To generate a secure API key, we need to create a random string of characters and hash it using a secure hash function.

Here's an example of how to generate an API key using hashlib:

```python
def generate_api_key(user_id: str) -> str:
    try:
        salt = private_salt
        key = user_id + salt
        generated_key = hashlib.sha256(key.encode()).hexdigest()
        #format the key to your needs
        formated_key = f"MV_SECRET_{generated_key}"
        return formated_key
    except Exception as e:
        raise e
```

### Saving the generated keys

Once we have generated a secure API key, we need to save it to our database so that we can use it for authentication later.

Assuming we have created a model for storing our API keys using SQLAlchemy's Base class, we can create a new instance of the model and set its attributes to the values we want to save. Here's an example:

```python
def save_generated_key(user_id:str,db:Session):
    try:
        check_existence = db.query(UserApiKeys).filter(UserApiKeys.user_id==user_id).first()
        
        if check_existence:
            return check_existence.secret_key

        user = db.query(Users).filter(Users.user_id==user_id).first()
        
        secret_key = generate_api_key(user_id=user_id)
        
        to_hash = hash_api_key(secret_key)
        
        public_key = f"MV_PUBLIC_{to_hash}"

        save_key = UserApiKeys(
            user_id=user.user_id,
            secret_key=secret_key,
            public_key = public_key
        )
        db.add(save_key)
        db.commit()
        return {"secret_key:":secret_key,"public_key":public_key}

    except Exception as e:
        print(e)
        return False
```

### Verifying the Keys

Once we have generated and saved an API key, we need to verify it when it is sent with an API request. To do this, we compare the hashed API key sent with the request to the hashed API key we have stored in our database.

Here's an example of how to verify the API key:

```python
def verify_api_key(db:Session,token:str):
    try:

        key = db.query(UserApiKeys).filter(UserApiKeys.secret_key==token).first()

        if not key:
            raise HTTPException(400,detail="Keys do not exist")
        #verifying extracting the values after the prefix
        input_str = str(key.public_key)
        prefix = "MV_PUBLIC_"
        prefix_index = input_str.find(prefix)

        if prefix_index >= 0:
            value = input_str[prefix_index + len(prefix):]
            print(value)
            verified_key = value
        else:
            print("Prefix not found in input string")
        
        to_verify = verify_hashed_key(api_key=key.secret_key, hashed_api_key=verified_key)

        if not to_verify:
            raise HTTPException(400,detail="Keys do not match")
        user  = db.query(Users).filter(Users.user_id==key.user_id).first()

        return user
    except Exception as e:
        print(e)
        return False
```

### **Setting the Header with FastAPI**

```python
def api_key_header(authorization: str = Header(...)):
    if not authorization.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="Invalid authorization header")
    token = authorization.split(" ")[1]
    return token
```

### **Example Use Case**

API key authentication with FastAPI can help restrict access to certain resources or functionality based on the user or application making the request. By generating and requiring an API key, you can ensure that only authorized users or applications can access certain parts of your application.

Here is an example use case

```python
app = FastAPI()

@app.get("/user")
async def get_cureent_user(authorization:str = Depends(api_key_header),db:Session = Depends(get_db)):
    """
        send the secret_key as bearer token
        e.g {"authorization":" Bearer secret_key"}
    """
    check_key = verify_api_key(db=db, token=authorization)
    if check_key != False:
        #do every operation down
        return check_key
    raise HTTPException(status_code=401, detail="Not Authorized")
```

### **Conclusion**

In this article, we have seen how to implement API key authentication with FastAPI. By generating and requiring an API key, you can ensure that only authorized users or applications can access certain resources or functionality in your application. We have covered the steps involved in generating, saving, and verifying the API key using hashing and FastAPI's `Header` dependency.