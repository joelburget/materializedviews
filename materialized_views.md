- Feature Name: Materialized Views, Change propagation,
- Status:
- Start Date:
- Authors: Arjun Narayan
- RFC PR: TBD
- Cockroach Issue: None.

# Summary

Materialized views are a powerful feature for running analytics
queries on a database. This RFC proposes a framework for executing
materialized views on top of the existing distributed SQL (DistSQL)
infrastructure.


# Motivation

Consider the following use case: we wish to implement an email
client. There is a table "emails", with the following schema:

    CREATE TABLE emails (
        id INT SERIAL PRIMARY KEY,
        receiver INT FOREIGN KEY users.id NOT NULL,
        sender STRING NOT NULL,
        read BOOL NOT NULL,
