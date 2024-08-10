# Design

## Overview

Graph-based authorization systems are a powerful tool for managing access control in complex environments. By representing permissions and relationships as nodes and edges in a graph, these systems can provide fine-grained access control, support dynamic authorization decisions, and enable efficient policy management.


## When to Use This Blueprint

There are a number of different scenarios where a graph-based authorization system may be beneficial. Some common use cases include:
- Managing complex access control requirements
- Integrating across multiple systems and data sources
- Supporting dynamic authorization decisions

## Why use Graph 

Policy as data
Fast Authorization for Complex Data




# Components

Ping Directory
PingOne Authorize
PingOne
Graph Database (TinkerPop, JanusGraph, etc.)

## Principles

- There is not one-size-fits-all solution for authorization. The best approach will depend on the specific requirements of the organization.
- Role based access control (RBAC) is a common approach to authorization, but it is not always right for every situation. Organizations should consider other models, such as attribute-based access control (ABAC) or graph-based access control, to find the best fit for their needs.
- Authorization should be designed with scalability and performance in mind. As the organization grows and the number of users and resources increases, the authorization system should be able to handle the increased load without sacrificing performance.
- Graph is not a silver bullet. It is a powerful tool, but it is not always the right choice for every situation. Organizations should carefully consider their requirements and evaluate whether a graph-based authorization system is the best fit for their needs.
- Start with your data sources, then build your graph. The quality of the data in the graph will determine the quality of the authorization decisions that can be made. Organizations should ensure that their data sources are accurate, up-to-date, and complete before building the graph.
- Dont master data in the graph. The graph should be a representation of the data in the organization, not the master copy. Organizations should ensure that the data in the graph is kept in sync with the data sources and that changes are propagated to the graph in a timely manner.
- Consistency is key. Organizations should establish clear policies and procedures for managing the graph and ensure that these are followed consistently across the organization. This will help to avoid errors and inconsistencies in the graph that could lead to incorrect authorization decisions.

## Definitions

- Entitlements: 
- Consent: 
- Permissions:
- Relationships:
- Nodes:
- Edges:
- ABAC:
- RBAC:
- ReBAC:
- LDAP:
- GQL: