# Pricing Service Concept

## Basic Function

This service maintains up-to-date exchange rate data between various currencies from various sources.

## Basic Requirements

- For each active currency pair the service will need to regularly request data from one or more 3rd party sources

- It should be easy to add new data sources

- Historical exchange rate data should be preserved

- It should be easy to activate / deactivate currency pairs

- It should be able to send alerts when a currency pair from a specific source is no longer valid
