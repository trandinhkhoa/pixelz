# Pixelz Checkout & Production Push — Problem Statement

## Background
At Pixelz, we help ecommerce studio professionals beat deadlines by providing reliable, AI-powered retouching at scale.

We have a **Portal application** that allows customers to:
- Create orders
- Pay for orders
- Handle recurring billing

---

## Scenario
Clients need to **check out their orders** (pay for the orders) so the orders can be pushed to our internal **Production system**.

### Expected behavior
When a client successfully checks out an order:

1. The system must confirm payment success
2. The system must push the order to the internal **Production system**
3. The client must receive a **successful payment email**

---

## Example User Flow
1. An **Art Director** goes to the system
2. The Art Director searches for orders they have already created
3. The Art Director finds an order by **order name**
4. The Art Director checks out the order
5. If payment is successful, the order is pushed to the **Production system**

---

## Functional Requirements

### Order Search
- Orders must be searchable / filterable by **name**

### Checkout / Payment
- Customers must be able to check out existing orders
- The system must detect whether checkout/payment is successful

### Production Push
- If checkout is successful, the system must call the internal **Production system**
- The internal system must update the order state so the order can be processed

### Email Notification
- The client must receive an email if payment is successful

---

## Design Intent
We plan to build our own backend(s) to orchestrate the full checkout flow:

- From receiving the checkout request
- Through payment processing
- Until the order is successfully sent to the Production system

---

## Key Architecture Goals

### Support multiple PSPs over time
The system should be designed to support multiple **Payment Service Providers (PSPs)** in the future.

### Avoid duplicate processing
The system must prevent:
- Paying the same order multiple times
- Sending the same paid order to Production multiple times
