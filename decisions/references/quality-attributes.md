# Quality Attributes

Quality attributes are the properties of a system beyond what it does: how fast, how reliable, how secure, how easy to change. Decisions are mostly trade-offs between them. This file helps identify which matter and how they conflict.

## Contents

1. The attributes
2. Questions to identify what matters
3. Common tensions
4. Making attributes concrete

## 1. The attributes

Based on ISO/IEC 25010 and common architecture practice:

| Attribute | Question it answers |
|---|---|
| **Performance** | How quickly does it respond, and how much work can it handle? |
| **Scalability** | Can it handle growth in users, data, or load without redesign? |
| **Availability** | Is it up when people need it? |
| **Reliability** | Does it keep working correctly when parts fail? |
| **Security** | Are data and access protected? |
| **Maintainability** | How easily can it be understood, changed, and extended? |
| **Testability** | How easily can its behaviour be verified? |
| **Simplicity** | How few concepts and moving parts does it have? |
| **Operability** | How easily can it be deployed, monitored, and debugged? |
| **Usability** | How easily can people use it? Includes accessibility. |
| **Compatibility** | How well does it work with existing systems and clients? |
| **Portability** | How easily can it move between environments or providers? |
| **Cost** | What does it cost to build, run, and maintain? |
| **Time to market** | How quickly can it be delivered? |
| **Compliance** | Does it meet legal and regulatory requirements? |

## 2. Questions to identify what matters

Ask the person you're working with, or find in the project's documentation:

- **Who uses it, and how many?** Ten internal users and ten million public users need different designs.
- **What happens if it's down for an hour?** An inconvenience, lost revenue, or real harm?
- **What happens if data is wrong or lost?** Recoverable, or serious?
- **How sensitive is the data?** Public, internal, personal, financial, health?
- **How fast must it respond?** Is anyone waiting on it in real time?
- **How quickly will it grow?** In users, data, and features?
- **How often will it change?** A stable tool or a product under active development?
- **Who maintains it?** One person, a small team, several teams? With what skills?
- **What are the constraints?** Budget, deadlines, required technologies, regulations?

The answers usually make two or three attributes clearly the most important.

## 3. Common tensions

| Tension | Explanation |
|---|---|
| Performance vs. maintainability | Optimised code is often harder to read and change. Optimise only where measurement shows a need. |
| Scalability vs. simplicity | Distributed systems scale further but are much more complex to build and run. |
| Flexibility vs. simplicity | Configurable, generic designs handle more cases but are harder to understand. |
| Consistency vs. availability | In distributed systems, you can't always have both during network failures. Decide which matters more for each kind of data. |
| Security vs. usability | Stronger controls add friction. Find the protections with the best balance, such as passkeys over complex password rules. |
| Time to market vs. quality | Shortcuts deliver sooner but cost later. Take them deliberately (see `evaluating-options.md`). |
| Cost vs. reliability | Redundancy and backups cost money. Match investment to the cost of failure. |
| Build vs. buy | Building gives control; buying saves time but adds dependency and cost. |

There's rarely a universally right answer. The right answer follows from which attributes rank highest for this system.

## 4. Making attributes concrete

Vague goals ("it should be fast") can't guide decisions. Make them measurable where it matters:

| Vague | Concrete |
|---|---|
| Fast | 95% of API requests complete within 300 ms at expected load |
| Highly available | Available 99.9% of the time each month |
| Scalable | Handles 10× current traffic without redesign |
| Secure | Meets the OWASP ASVS level the project targets |
| Maintainable | A new developer can make a typical change within their first week |

Concrete targets show when a design is good enough, and prevent over-engineering for goals nobody needs.