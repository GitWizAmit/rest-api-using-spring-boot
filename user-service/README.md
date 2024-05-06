---
description: >-
  This section deals with developing the User Service which will be dealing with
  the users in the system.
---

# User Service

### The process of development in a nutshell. <a href="#the-process-of-development-in-a-nutshell" id="the-process-of-development-in-a-nutshell"></a>

1. Define the service to handle CRUD operations for the User.
2. Refactor the service to be independent of data transactions.
3. Implement rest controllers for the clients to interact with the system.
4. Scale the repository to use a persistent database.

### CRUD: Create Read Update Delete <a href="#crud-create-read-update-delete" id="crud-create-read-update-delete"></a>

Before any server side application is developed we should look towards establishing the model definition so that it would be easier to implement. The models can initially be minimal and has full freedom to be expanded later but the minimum design level fields must be defined on top of which the architecture would be built. We'll be defining them now.

Once we have the service refactored to begin interacting with the world, we will be defining the API endpoints, the request and response bodies for each endpoint. These definitions will facilitate the clients into integrating our system for their use. Users interact with any service from a user-facing client application which converts the user's instructions into API commands and vice-versa.

### Test driven philosophy <a href="#test-driven-philosophy" id="test-driven-philosophy"></a>

The three laws of TDD:

* Write production code only to pass a failing unit test.
* Write no more of a unit test than sufficient to fail (compilation failures are failures).
* Write no more production code than necessary to pass the one failing unit test.
