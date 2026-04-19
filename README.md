# CS4700 Project 6: DNS Server by Joshua Sharma and Kevin Mo

## Project Description
This DNS server works like a real recursive DNS resolver. When given a question by a client, the server can respond with answer fit for their question type. There are two
major branches of how questions are resolved.

### Questions under Authoritative Domain
If the given query asks about a domain that happens to be in the server's authority, then resolving becomes simple:
- Intialize server's authoritative domain records by parsing a given zone file
- Find out if potential answers can be found in server records
- If CNAMES are encountered, resolve by following the CNAME chain until we get to the true name of the domain (if CNAME target is outside zone then just return what server has so far)
- If errors/problems are encountered (ex. question's target domain didn't exist in local records, question type's answer is not found), then respond with approriate rcode based on assignment description

### Questions outside Authoritative Domain
If the given query asks about a domain that is outside the server's authority, then:
- Check if recursion flag RD is set to 1. If client doesn't want recursion, simply return SERVFAIL
- Start recursively resolving where at each step:
- Follow CNAME chains until we get the target domain name, restarting from the root server if we find the true domain name
- Follow referrals by collecting NS servers (using glue records to get IP if present, otherwise resolve NS's IP)
- Find a final answer if the response has a record matching the qname and qtype of the client's question
- Build the response for the client + copy flags from client request and clear truncate flag

### Recursive Resolver Implementation
Each iterative query starts at the closest server we know about. We check the cache for the closest parent zone with NS records, and if nothing is cached we fall back to the root IP from the command line. Then we loop:
- Send the query with RD = 0 over UDP port 60053
- Drop any records outside the responding server's bailiwick
- Cache the surviving records (only on NOERROR)
- If we got the answer, return it
- If we got a CNAME, recurse on the target with the chain accumulated
- If we got a referral, pick the next servers from glue A records, or resolve the NS hostname if no glue was provided

### Caching
Our cache is a thread-safe dict keyed by (name, qtype). Each entry stores the original RR and its expiry time. We skip TTL = 0 records, dedupe by rdata, evict expired entries on get, and return RRs with TTLs decremented. NS answers served from cache also pull their A glue from the cache so the additional section still matches.

### Bailiwick Checking
Every upstream response is filtered before we cache or use it. A record is in bailiwick if its name equals or is a subdomain of the zone we just queried. The root server is in bailiwick of . so its records always pass. After filtering we re-sync the header counts.

### Concurrency
The processes above can be done concurrently if there are multiple client requests incoming. Each incoming packet is handed off to a new thread so the main loop can keep listening. The cache uses a lock so threads can put/get safely.

## Challenges

### Figuring out how to concurrently process requests
After doing research for how to enable concurrent processing of requests, we found multiple solutions such as asynchronous I/O. However, we decided that the cleanest way to implement concurrency was to use multithreading, where some threads are allowed to wait. Threading allows the program to start more threads for request processing.

### Filling gaps in recursive resolving 
Since DNS server responses may vary wildly, we had to thoroughly check that all cases were accounted for. Here are all the cases we encountered and guarded for:
- Multiple questions sent by client at one time
- Malformed DNS
- Timeouts or failed parsing
- Answering with given record in AUTHORITY if client question is for the given rtype
- CNAME chain resolving
- Client sets RD = 0 so no recursion
- Resolving fails because no answer can be found for a given question
- non NOERROR rcode from a server being queried
- successful answer found

### Getting the cache to behave like a real resolver
Caching looked simple at first but had a few parts. We had to count TTLs down between hits, never cache error responses, and make sure NS answers from cache still carried their glue A records. Filtering by bailiwick before caching was important so an upstream couldn't push records for zones it didn't own.

### Features of Design We Consider Good
- Clear split between questions for domains in server authority vs outside
- Handling all header matching for returning answers to clients
- Concurrency being handled via multithreading
- Organization of code into smaller helpers
- Cache and bailiwick logic kept in their own helpers so the resolver loop stays readable

### Testing
- After every major step mentioned in the Assignment Description's Implementation Strategy, we ran the relevant tests. We only moved on if all tests for a given step passed. 
- After completing the entire project, we ran the test suite fully and also used the automatic tests on Gradescope to double check.
