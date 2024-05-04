# Design decisions

Before embarking upon any adventure, one must have a bird's eye view of what they want to achieve and some points along their pathway. You might think you know a substantial portion at the beginning itself, no matter what you're bound to fill the gaps as you go along. Even if you have no clue at the beginning, you will find them on your journey, just sit tight and keep at it.

Software projects have clear separation in terms of implementation technique and are divided into server side and client side. These two entities must be able to talk with each other clearly and as humans design both side of things, it would be nice if humans can also understand the communication without going into any sort of assembly language equations which only the computers know.

The simplest mechanism with which servers & clients interact is using REST APIs (Representational State Transfer) and is usually facilitated using JSON (Javascript Object Notation) Objects.

When a server side service is being defined, various players comes into the equation. There are certain practices that the industry follows to developed maintainable and scalable systems. We would want the system to work at fast rates and also the code to be easily changeable with concentrated effort on smaller parts without disturbing the whole system.

The system has 3 components to begin with.

1. Business Domain Logic
2. Rest Controllers
3. Persistent Database

Business domain logic acts as the middle man between other components. Rest Controllers are user facing and database has to be kept away from the user at all times. User should never be able to take actions on the database without going through the domain logic. It makes sense to keep the components decoupled with each other and can be swapped at any time.

Based on the principle of separation of concerns, the logical point to begin development seems to be the domain and write tests around these components. We will be using interfaces in injecting dependencies so as to hide away the concrete logic and with the freedom to change the concrete logic at any time without disturbing the other components.

To get the domain logic working, we would require a domain model with which we can work. The models are simple classes which specifies the information about the entities in the system. We must think on how these components can reflect the requirements. We can begin the initial development with storing the information in an in-memory database which should easily be swapped with a persistent database later.

Once we have the domain service working, we will looking to expose controllers with which users can interact with the system. These controllers can be used by web and mobile clients to perform actions in a sensible fashion. The controllers act as a wrapper between the external world and domain service. It also acts as a security layer and invalid requests can be prevented to reach the domain service in the controller itself.

Having an in-memory database means that it will start up fresh and without any data every time the service launches. It's not feasible to keep the system running at all times. Any service outage will delete all the data in the system. Also, there can be too many service requests if queried from an in-memory database and the service can be denied fairly quickly. The users wouldn't like that and the application won't be scaled. We would be replacing that database with a persistent one and resolve these issues.
