### Saving Articles
When a user saves an article, use the `ArticleMeta` interface and specify the 
`savedDate` attribute to the date the article was saved.
```typescript
interface ArticleMeta extends Article {
  savedDate: Date;   // the date an article was saved by user
}
```