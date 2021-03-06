
+++

+++
# Onion Architecture

Onion Architecture is a software application architecture that adheres to the SOLID principles. It uses the dependency injection principle, and it is influenced by the Domain Driven Design (DDD) and functional programming principles.

Concerns are the different aspects of software functionality: the "business logic" of software is a concern, and the interface through which a person uses this logic is another concern.

The separation of concerns is keeping the code for each of these concerns separated. Changing the interface should not require changing the business logic code, and vice versa.

The repository mediates between the data source layer and the business layers of the application. It queries the data source for the data, maps the data from the data source to a business entity, and persists changes in the business entity to the data source. A repository separates the business logic from the interactions with the underlying data source.

