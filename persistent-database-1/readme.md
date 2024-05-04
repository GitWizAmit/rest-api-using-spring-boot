# Persistent Database

For the ability to scale a system, we require the database to execute concurrent queries and persist data across service sessions. We'll be using Postgres as our persistent database.

We have tests written for the controller layer and they are acting as integration tests for the system. It'd nice to keep that going and have the database calls also tested instead of mocking the repository calls. It'll also gives the confidence to test the sql statements making our implementation more robust.

