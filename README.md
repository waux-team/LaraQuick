# LaraQuick

LaraQuick is a Laravel package that provides a framework for building and managing artifact services in Laravel applications. It offers modular, extensible services for various functionalities, ensuring security, normalization, and best practices like Third Normal Form (3NF) in database design.

## Key Features

- **Modular Services**: Includes services for user management, repositories, and extensible support for additional services.
- **Normalized Schema**: Designed to eliminate redundancy and enforce the principle of least privilege, isolating sensitive data into dedicated tables.
- **Extensible Architecture**: Easily add new services and functionalities as needed.
- **Laravel Integration**: Seamlessly integrates with Laravel applications for easy setup and use.

## Database Design

The package includes a detailed database design report (see `Docs/User.md`) outlining a normalized schema for user-related tables, such as:
- `users`: Anchor table for user identity
- `user_contacts`: Contact information
- `user_credentials`: Authentication data
- `user_addresses`: Multiple addresses per user
- `user_sessions`: Session management
- `user_locations`: Location tracking

This design serves as a template for other services, ensuring consistency and security across the application.
