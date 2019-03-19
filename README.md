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
