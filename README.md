**Problem:** We want to easily build features that show data and render graphs. Currently, getting complex data ranges from GraphQL is complicated and therefore time-consuming.

**Solution:** Write React components that, using H1QL, query the database directly to get their data needs. H1QL is a safe subset of SQL and enforces strict access control to only expose data the requester can see. Engineers can quickly build reporting features as they are already familiar with the power of SQL and don’t have to invest in making it secure. H1QL is a typed language, this enables the component to know what type of graphs or other display options can be used.

# Usage

## In React
H1QL is a React component that requires a query that will be used to render its children.

```
<H1QL query=“SELECT COUNT(*) as count, MAX(id) as max FROM users”>
   ({ rows }) => {
      <p>
        Number of users: {rows[0].count}<br />
        Latest id: {rows[0].max}
      </p>
   }
</H1QL>
```

## In Rails
Any query executed within an H1QL block will be automatically secured.

```
H1QL.new(requester: User.first) do
  puts User.where(‘id > 42’).count
end
```

## Inner workings of H1QL
H1QL is a subset of SQL, it only supports operations that can be executed safely. For example, only non-mutative operations are supported and it doesn't allow operations that need direct file access. The H1QL engine will only return data the user is authorized to see. This is where H1QL really shines, restrictions on data access are centralized and any query can be executed safely no matter the source of the request. 

```
     +                H1QL Engine                                                       +
User |               +---------------------------------------------------+     Database |
     |               |                                                   |              |
     |               |  tokenization  validation  transform   to sql     |              |
     |  H1QL Query   |   & parsing                                       |  SQL Query   |
     | +-----------> |       +            +           +          +       | +----------> |
     |               |       |            |           |          |       |              |
     |               | +---> | +---+----> | +---+---> | +---+--+ | +-+-> |              |
     |               |       |     |      |     |     |     |    |   |   |              |
     |               |       +     |      +     |     +     |    +   |   |              |
     |               |             +            +           +        +   |              |
     |               |          SQL AST      (unsafe)     (safe)    SQL  |              |
     |               |                         H1QL        H1QL          |              |
     |               |                                                   |              |
     |               +---------------------------------------------------+              |
     |                                                                                  |
     |                                                                        Response  |
     | <------------------------------------------------------------------------------+ |
     |                                                                                  |
     +                                                                                  +                                                                                                  |
```


