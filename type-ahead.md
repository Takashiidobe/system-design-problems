# Design a Type-Ahead service

Imagine you have to design the backend for a typeahead system.
Typeahead should return a list of 3 top suggestions, based on what the
user types in the search box.

The suggestions are ordered according to the frequency and recency of
their appearance in user searches.

This must be in real time -- it should be fast and scalable.
