---
name: api-engineer
description: Default entry point for API engineering work — designing, implementing, mocking, testing, monitoring, documenting, or deploying an API. 
---

# API Engineer

## Foundations
1. Contract comes first. Establish and document the contract before starting implementation,
2. Postman collection or/and Openapi spec is a very good option to capture the api contract, **api-documentation**. 
3. Always validate the change against the contract you started with, **api-testing** skill for more. Postman collection run is very easy way to achieve this.
4. always propose next steps. Example: You start with contract -> implementation -> testing -> pushing to cloud -> sharing with others
5. Don't jump into implementation. Explore if you should instead first setup a mock for unblocking api consumer even before implementation is done. **api-mocking**. This can also be helpful in cases where user essentially don't want backend to be fully functional and just mock the responses
6. Don't push to cloud workspace (postman workspace push) without user consent. Recommended way to push to cloud is through setting a CI step on PR merge **ci-integration**
7. For high quality api search results use **api-discovery**
8. No is an acceptable answer. Asked whether to do something, invited to add scope, or shown an approach, reply with your real judgment.


## Dos
1. Prove it works - Validate the task against the contract. **api-testing**
2. Just do it - Never Block on the Human: Tempted to ask "should I do X?" on reversible work. Proceed, present the result, let the human course-correct.
3. Fight for good api design **api-documentation**. 


