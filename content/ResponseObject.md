# GraphResponse objects in Microsoft Graph SDKs

## Objectives

Provide a GraphResponse class that contains the entire response sent to the client an enables lazy deserialization <!-- maybe even custom deserialization -->.

## Problem

The generated client libraries don't return the entire response envelope. The response body is deserialized into a returned object. This results in data loss of information in response headers, status codes, and status messages which may be actionable by the client.

## Requirements

- GraphResponse MUST provide the HTTP response headers, status codes, and the raw response body.
- It SHOULD provide a method to lazy deserialize the response body into our generated models or another object type based on whether a custom deserializer is provided.
- It MAY support providing a custom deserializer to override the default deserializer.

## Performance Considerations

Lazy deserialization should improve performance for scenarios where customers do not need to immediately access the returned objects.

Custom deserializer support can enable customers to craft a deserializer optimized for a specific response.


## Current GraphResponse objects in Microsoft Graph SDKs

### Javascript

The Javascript SDK exposes the native [Response](https://developer.mozilla.org/en-US/docs/Web/API/Response) object when selecting the RAW ResponseType option.

### .Net

> TODO: Currently not supported. Needs proposal.

### Java

> TODO: Currently not supported. Needs proposal. Should be able to use the same pattern as .Net.

### PHP

The PHP SDK returns a Graph model (or collection) or a GraphResponse object, depending on whether a returnType is specified. This means that the returnType information is not supplied to the GraphResponse. The response body is set as a JSON array on GraphResponse. The only work I'd suggest here is that we add an option to pass the returnType to the GraphResponse and add a lazy deserialize method so it is easy to get the Graph models.

### Objective-C

<!-- TODO: needs statement -->

## References

<!-- TODO: Add these reference. -->

---
<!-- TODO: Write up for .Net goes in separate doc -->

The generated .Net client library either returns: 1) the response content as an object or collection object as the result of a successful call to Microsoft Graph, or 2) a ServiceException in the case the call fails.

We consider this a design flaw as the client library swallows useful information such as the response headers and the http status code. Mitigation such as stuffing the information in AdditionalData is insufficient for all scenarios. For example, there is no AdditionalData on Stream objects, and Delete operations return void.

Related to:
https://github.com/microsoftgraph/msgraph-sdk-dotnet/issues/457
https://github.com/microsoftgraph/msgraph-sdk-dotnet/issues/361

## Options for exposing a GraphResponse object

<!-- The generated request builders return request interfaces based on IBaseRequest while the Request objects descend from BaseRequest. -->

**Option 1. Set on IBaseRequest**

We'd specify methods on IBaseRequest that would send the request and return the response as a GraphResponse object.


```csharp
public interface IBaseRequest
{
    // Ugh, horrible names
    Task<GraphResponse<T>> GetWithResponseObjectAsync<T>();
    Task<GraphResponse<T>> CreateWithResponseObjectAsync<T>(NewObject: Entity);
    Task<GraphResponse<T>> PostWithResponseObjectAsync<T>(NewObject: Entity);
    Task<GraphResponse<T>> UpdateWithResponseObjectAsync<T>(UpdatedObject: Entity);
    Task<GraphResponse> DeleteWithResponseObjectAsync();
}
```

_Pros_

* Be implemented in a single place: `BaseRequest` in Microsoft.Graph.Core.
* No metaprogramming, no T4 templates code to maintain.
* Could be used to not only target entities, but could be used to target complexType structural properties. No need for custom request builder


_Cons_

* Would not respect capability annotations. These methods would be available on all requests regardless of whether it applies. API signature pollution.
* Customers would need to know the return types. Note that the corresponding generated method's return type would be a hint to tell our customers what this return should be, with the exception of Delete methods. For example:

```
Task<UserCollectionPage> page = await client.Users.Request().GetAsync(); // Intellisense here can inform for the next line.

// Must know the generic type
Task<GraphResponse<UserCollectionPage> page = await client.Users.Request().GetWithResponseObjectAsync<UserCollectionPage>();
```



**Option 2. Generated on the $Request objects**

We propose that we create parallel *Async functions to enable the GraphResponse object functionality. We will generate requests with the following public API signatures:

* `GetWithResponseObjectAsync(): GraphResponse<T>, where T is the return type`
* `CreateWithResponseObjectAsync(NewObject: Entity) : GraphResponse<T>, where T is the return type`
* `PostWithResponseObjectAsync(NewObject: Entity) : GraphResponse<T>, where T is the return type`
* `UpdateWithResponseObjectAsync(UpdatedObject: Entity) : GraphResponse<T>, where T is the return type`
* `DeleteWithResponseObjectAsync() : GraphResponse`

**Example**


`GraphResponse<UserCollectionPage> response = await graphServiceClient.Users.Request().GetWithResponseObjectAsync(cancellationToken);`

`GraphResponse<User> response = await graphServiceClient.Me.Request().UpdateWithResponseObjectAsync(patchUser, cancellationToken);`

<!-- TODO-Q: Should we have late deserialization? -->

### GraphResponse

The GraphResponse object can wrap the HttpResponseMessage object so that customers can use the familiar HttpResponseMessage object. We will provide a method to provide late deserialization for the object returned in the response. GraphResponse will be part of Microsoft.Graph.Core.

```
/// For DELETE where there is not object returned. We could just use the generic object for DELETE
public class GraphResponse : HttpResponseMessage
{
    private ResponseHandler responseHandler;

    public GraphResponse(ResponseHandler rh)
    {
        responseHandler = rh;
    }

    // No response body.
}

/// For responses with response object.
public class GraphResponse<T> : GraphResponse
{
    public GraphResponse(ResponseHandler rh) : base(rh)
    { }

    public T GetResponseObject<T>()
    {
        // Use the response handler to return us the deserialized response
    }
}

/// For deserializing paged responses.
public class GraphResponse<U,T> : HttpResponseMessage where T : ICollectionPage<U>
{}
```



### Scenarios

* Deserialize entity.
* Deserialize complex type.
* Paging for delta and nextlink.
* Batch



The generator will generate the type for T so these requests

GraphResponse.Content
GraphResponse.HttpHeaders
GraphResponse.StatusCode



Why in the generated files?
* Use generator to get capa annotation generation support.
* Cons: T4 template code to maintain.


Things we considered.
- We can't just use AdditionalData as getting Stream and DeleteAsync don't have a Graph content response.

* We add overload to existing generated functions. GetAsync(WithResponseObject: boolean); where we would deprecate existing overloads to state that this is the new way. *
* Can't use the existing AdditionalData as DeleteAsync() doesn't return an object.




## Opportunities

We realize that this is an op


Opportunity to implement stuff from our proposed "breaking moment" a simplified http provider. This would also allow us to tryout enabling custom serlializer as well.

We could release this as an experimental preview.
After prview, we de-emphasize *Async classic methods and update the snippet generator to use this form. Deemphasis will be done via blog post and /// comment.
We instrument the generated methods so we discover usage.
We deprecate when we hit a threshold. Deprecation will be done via blog post and /// comment.
We can delete at another threshold. The cool thing is that they will still get the latest functionality in the service libraries since they are decoupled.

## Deemphasis to deprecation to breaking change plan