---
layout: post
title: "#4 - Multi-Timezone: Things to Know"
date: 2021-08-23T14:00:00+0700
slug: multi-timezone-things-to-know
categories: [other]
image: multi-timezone.jpg
tags: [
    'backend',
    'timezone',
    'multi timezone',
]
---

> TLDR;
> - Set the server's timezone to UTC.
> - Synchronize the timezone between the server and the database.
> - Use ISO8601 + RFC3339 format when returning datetime to clients.
> - Convert datetime to UTC before saving it to the database.

## UTC as the Server's Timezone

When a system or project needs to support multiple timezones, the first
consideration is the server's timezone. Depending on the project's scale,
the servers may be located in different timezones. Therefore, it's essential
to choose a default timezone to avoid inconsistencies when the servers work
together. The recommended default timezone is Coordinated Universal Time (UTC) (+0).

> Why UTC?

UTC stands for Coordinated Universal Time
([en.wikipedia.org/wiki/Coordinated_Universal_Time](en.wikipedia.org/wiki/Coordinated_Universal_Time)).
It is an international standard that is widely used by various systems and
supported by datetime libraries. Many foreign cloud providers also use UTC as
the default timezone when initializing virtual private servers (VPS). Choosing
UTC is a reasonable choice as it simplifies the configuration process. During
development or testing, the project team can easily convert datetime values
manually when needed.

> For more information, refer to: Always Use UTC Dates and Times:
[Always Use UTC Dates And Times](https://kylekatarnls.medium.com/always-use-utc-dates-and-times-8a8200ca3164)

## Synchronize Timezone Database with the Server

If the database runs on the same server as the application code, there is no
need for concern. However, when the database is located on a separate server,
it is advisable to synchronize the timezone between them. This ensures that the
application avoids errors when retrieving and setting data. Additionally, popular
database management systems like MySQL often have a default timezone set to UTC,
which aligns with the recommended server timezone.

## Use ISO8601 + RFC3339 Format

> Client: What about the clients in different timezones?
> 
> Server: Don't worry! The client only needs to send its timezone to the
server, and the server will handle the conversion of datetime values accordingly.
> 
> Client: ...
> 
> Server: If you prefer another approach, we can explore alternative solutions.

ISO8601 and RFC3339 are widely recognized standards for representing date and
time information. ISO 8601 is an international standard for the interchange of
date- and time-related data ([ISO_8601](https://en.wikipedia.org/wiki/ISO_8601)),
while RFC3339 is an Internet Engineering Task Force (IETF) document that
defines a date and time format for internet protocols
([RFC3339](https://www.ietf.org/rfc/rfc3339.txt)).

Both standards are similar, with minor differences in some aspects. To
accommodate both standards, we can combine them and use the following format:

```text
2021-08-23T14:00:00+0700
```

If milliseconds are required for specific cases, the format can be extended as
follows:

```text
2021-08-23T14:00:00.345+0700
```

Note: You may encounter the format 2021-08-23T14:00:00Z, where Z denotes
UTC (+0), which is equivalent to +00:00. However, it's important to note that
many software engineers may not be familiar with this format, leading to
potential misunderstandings. Hence, it's recommended to limit the use of this
format.

By setting the default timezone to UTC, synchronizing the timezone between the
server and the database, and converting datetime to UTC before storing them in
the database, you can ensure consistent and accurate data, reducing the risk of
logic bugs.
