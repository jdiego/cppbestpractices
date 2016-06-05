# Resource Management 

## Interfaces

An interface is a contract between two parts of a program. Precisely stating what is expected of a supplier of a service and a user of that service is essential. Having good (easy-to-understand, encouraging efficient use, not error-prone, supporting testing, etc.) interfaces is probably the most important single aspect of code organization.

## Pointers in Interface
* Pointers in interfaces:
  * Should only be used to represent a single item
  * Shoud only be used to represent a single `optional` item 
* Consider using Guideline Support Library `span` class
* Consider using `optional` for optional parameters
* Use the tools available: C++ core checker
