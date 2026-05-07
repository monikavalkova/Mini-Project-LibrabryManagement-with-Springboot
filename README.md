# Library Management System — Spring Boot

A simple Library Management System built with Spring Boot and JSP. It supports full authentication and CRUD operations for books and members, with in-memory data storage.

---

## Getting Started

### Prerequisites
- Java 8
- Maven (or use the included `./mvnw` wrapper)

### Run the app

```bash
./mvnw spring-boot:run
```

The app starts on **http://localhost:8088**

### Login

| Username | Password |
|----------|----------|
| admin    | admin    |

---

## How to Use

### Books
| What you want to do | Where to go |
|---------------------|-------------|
| See all books | `/show-listof-all-books` |
| Add a book | `/add-book` |
| Edit a book | `/edit-book` |
| Search for a book | `/get-book-details` |
| Delete a book | `/delete-book?id=<id>` |

### Members
| What you want to do | Where to go |
|---------------------|-------------|
| See all members | `/show-listof-all-members` |
| Add a member | `/add-member` |
| Edit a member | `/edit-member` |
| Search for a member | `/get-member-details` |

### Book Borrowing (Smart Book System)
| What you want to do | Where to go |
|---------------------|-------------|
| Borrow books (up to 3) | `/sml-main` |
| View current borrows | `/show-barrow-list` |
| Return a book | `/del-smartbs?name=<memberName>` |
| View full borrow history | `/show-total-history` |

---

## Build & Test

```bash
# Build
./mvnw clean package

# Run all tests
./mvnw test

# Run a single test
./mvnw test -Dtest=LbmApplicationTests
```

---

## H2 Database Console

The in-memory H2 database console is available at:
**http://localhost:8088/h2**

---

## Working with GitHub Copilot

This repo includes a Copilot instructions file at `.github/copilot-instructions.md`.
It contains architecture notes, key conventions, and gotchas that help Copilot give better suggestions in this codebase.

### Example prompts you can use
- *"Add a new field `book_isbn` to BookPojo and update the add-book form"*
- *"Add a search-by-author endpoint to BookController"*
- *"Add a new member field and update the edit member form"*
- *"Write a service method to check if a book is currently borrowed"*
