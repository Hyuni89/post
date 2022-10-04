## Dependency

```kotlin
implementation("org.elasticsearch.client:elasticsearch-rest-client:7.15.2")
```

## Sample

```kotlin
val restClient: RestClient = RestClient.builder(
		HttpHost(host, port, "http")
    ).build()
val res = restClient.performRequest(
		Request(
        	"GET",
            "/index"	// endpoint
        ).apply {
        	addParameter("size", 10)
            setJsonEntity(query)
        }
    )
```

## Query

query는 [공식 페이지](https://www.elastic.co/guide/en/elasticsearch/reference/current/rest-apis.html)에 잘 나와 있다. 하지만 index에 포함된 전체 documents를 가져오려고 했을때 어려웠던 점이 있어서 기록차 남긴다.  
```
GET /index/_search?size=100
```
이렇게 원하는 크기만큼 가져올 수 있다. 하지만 documents가 많아서 한번에 가져올 수 있는 양이 정해져 있으므로 일정한 수로 나누어서 가져와야 한다. 위와 같은 query를 날리면 항상 같은 document가 가져와지므로 이후에 가져와야 할 offset을 지정해야 한다.  
```
GET /index/_search?size=100&from=100
```
이렇게 되면 100번째 document 이후로 100개의 documents를 가져오게 된다. 간단하게 이렇게 할 수 있지만 여기에서 문제가 발생한다. 10,000개 이상의 documents를 가져올 수 없다. 가져오려고 하면 `result window is too large` 에러가 발생한다. `max_result_window` 설정을 바꾸어 이를 피할 순 있지만, 성능 문제를 일으킬 수 있으니 위와 같은 query는 간단한 검색에서 사용되어야 할 것이다.  

이를 해결하기 위해, `search_after`를 사용한다. `search_after`를 사용하기 위해선 정렬이 되어있어야 하는데, 정렬된 기준으로 `search_after`의 documents를 반환하기 때문이다.  
```
GET /index/_search?size=100
{
	"sort": [
    	{
        	"sortField1": {
            	"order": "asc"
           	}
      	},
        {
        	"sortField2": {
            	"order": "desc"
           	}
      	}
 	],
    "search_after": [ fieldValue1, fieldValue2 ]
}
```
위와 같이 사용할 수 있다. `fieldValue1`과 `fieldValue2`는 이전 query로 받았던 응답에서 `sort` 필드가 있는데 그 값을 그대로 넣으면 된다.  