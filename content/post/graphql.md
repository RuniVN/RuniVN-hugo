+++
draft = false
date = "2017-07-22T16:45:32+07:00"
title = "Single endpoint với GraphQL (backend Go)"

+++

GraphQL được tạo ra bởi Facebook vào năm 2012, production ready năm 2016 nhưng hiện tại vẫn khá ít công ty và developers Việt Nam sử dụng. Nhân tiện có một khách hàng muốn xài GraphQL nên team mình đã apply vào luôn. Sau khi xài xong thì thấy khá kute phô mai que nên muốn share ít kiến thức tìm hiểu được.

<img src="https://s3-ap-southeast-1.amazonaws.com/kipalog.com/lq5y5mw0h0_image.png" class="img-center">
# Problem

Để hiểu được concreate problem của GraphQL thì phải nói tới REST.
## REST

Chắc hẳn mọi người đều đã làm việc với REST rồi.
Có rất nhiều thứ để nói về REST, mình sẽ tóm gọn lại một cách dễ hiểu.
REST hay Representational state transfer là tập hợp những architecture principles, qui định cách thức clients interact với server, giúp application có thể manage tài nguyên.

Citizen của REST là "resource". Khi bạn muốn retreive dữ liệu resource thì bạn dùng GET, khi muốn insert thì dùng POST... kiểu vậy.

Một điểm quan trọng đó là REST stateless và cacheable.

<img src="https://s3-ap-southeast-1.amazonaws.com/kipalog.com/gggntnv456_image.png" class="img-center">

## REST drawbacks

Ứng dụng mình chia làm 2 phần, backend là Go và frontend là React. Mình bắt đầu gặp những vấn đề khi ứng dụng grows:

- Số lượng API endpoint quá nhiều, đôi lúc sẽ phải cầm cái id này chạy tới endpoint này kết quả, sau đó cầm tập id được response chạy tới endpoint khác để lấy cái thật sự cần.
- Rất khó để define ra được một chuẩn chung của dữ liệu trả về, vì đôi lúc với một model, client có thể lúc thì cần fields này, lúc thì cần những fields khác.
- Khi mà đổi một cái gì đó, ví dụ schema, thì không biết những chỗ nào bị impact.




# GraphQL for the sake

GraphQL là một internal project của Facebook, sau đó được opensourced. Nó là một query language cho phép clients construct resource mà bạn muốn server trả về.

Ví dụ:

```
{
  me {
    name
  }
}
```

Đoạn query trên muốn khi gọi API thì lấy tên của chính mình. Kết quả trả về:

```
{
  "me": {
    "name": "Runi"
  }
}
```

Bạn có thể nhận ra query của nó gần như là một JSON mà bị missing value, và kết quả trả về là JSON với format đúng như client requests.


# REST vs GraphQL
Thật ra so sánh 2 thằng này thì không hợp lý lắm, point ở đây là so sánh 2 thằng này "in practice".

Giống:

- Đều là send data over HTTP request.
- Core idea đều là resource.
- Output cuối cùng là JSON là như nhau.
- Đều có cách phân biệt giữa write/read data.

Khác:

- REST coupled giữa cách bạn define data và cách bạn retrieve nó. Ví dụ `/songs/1`.  GraphQL separates endpoint và cách bạn lấy data.
- REST định nghĩa những thông tin resource ở server, clients chỉ make a call. GraphQL cho phép client đưa lên một datashape, nhiệm vụ server phải trả về đúng thông tin như vậy.
- Khi bạn muốn fetch nhiều related data bạn phải gọi multiple request ở REST. GraphQL cho phép bạn traverse entry point để lấy data bằng single request. (hacky way ở REST là tạo 1 endpoint mới gom response lại, too painful).
- Thay đổi read/write ở REST bằng http method, GraphQL bằng query.


Well, tới đây chắc các bạn cũng có một quick overview về GraphQL rồi. Mình sẽ apply vào Go backend xem thế nào.

# Integrate backend Go

Mình đã thử dùng GraphQL trên elixir và Ruby. Phải thừa nhận là vì dynamic language nên code ... rất sướng tay.
Trên Go thì hơi trâu bò một chút.

**Bài toán**: Mình sẽ tạo một endpoint GraphQL, dùng nó lấy random một bài hát trong database, và tạo một song mới.

Mình sử dụng `github.com/graphql-go/graphql` làm GraphQL implementation.

Vì GraphQL idiomatic là chỉ dùng single endpoint, nên mình chỉ serve:

```
http.Handle("/graphql", graphHandler)
```

Trong đó

```
func  graphHandler(w http.ResponseWriter, r *http.Request) {
    var schema, _ = graphql.NewSchema(graphql.SchemaConfig{
    Query:    query,
    Mutation: mutation,
    })

	result := graphql.Do(graphql.Params{
		Schema:        schema,
		RequestString: r.URL.Query().Get("query"),
	})

	json.NewEncoder(w).Encode(result)
}
```

Trong GraphQL, **query = read và  mutation = write.**

Hàm trên có thể hiểu, graphql handler sẽ đọc vào một cái schema, parse query string từ client gửi lên để lấy datashape, sau đó query data từ nơi nào đó trả về.

## Query
Giờ mình sẽ construct query object để lấy bài hát.

```
var query = graphql.NewObject(graphql.ObjectConfig{
		Name: "Query",
		Fields: graphql.Fields{
			"song": &graphql.Field{
				Type: songType,
				Resolve: func(p graphql.ResolveParams) (interface{}, error) {
					song, err := getRandomSong()
					if err != nil {
						logrus.Errorf("failed to random song, err = %v", err)
						return nil, err
					}

					return song, nil
				},
			},
		},
	})



var songType = graphql.NewObject(graphql.ObjectConfig{
	Name:        "Song",
	Description: "Song contains some information",
	Fields: graphql.Fields{
		"id": &graphql.Field{
			Type:        graphql.ID,
			Description: "Song's id",
		},
		"title": &graphql.Field{
			Type:        graphql.String,
			Description: "The title of the song.",
		},
		"artist": &graphql.Field{
			Type:        graphql.String,
			Description: "The artist of the song.",
		},
	},
})
```

Có 2 thứ ở đây cần chú ý:

- Type của object bạn muốn trả về, đoạn code trên là `songType` qui định datashape mà client có thể query để lấy được.
- Resolve function:  Khi bạn viết một schema cho GraphQL, bạn phải viết resolve function cho nó. GraphQL execution engine sẽ invoke function này khi data thật sự được queried.

Như vậy với đoạn code trên, nếu không có gì xảy ra thì khi make request với query như sau:

```
query {
	song {
    	id
  		title
		artist
    }
}
```

sẽ được kết quả
```
{
    "song": {
        "id": 1,
        "title": "Chieu Hom Ay",
        "artist": "Jaykii"
    }
}
```

Match exactly với schema chúng ta vừa định nghĩa.

> Vậy còn nếu muốn random một bài hát, mà có theo `tag` do client gửi lên thì sao?

GraphQL hỗ trợ arguments, ta sẽ sửa lại code schema:

```
var query = graphql.NewObject(graphql.ObjectConfig{
		Name: "Query",
		Fields: graphql.Fields{
			"song": &graphql.Field{
				Type: songType,
                Args: graphql.FieldConfigArgument{
					"tag": &graphql.ArgumentConfig{
						Type: graphql.String,
					},
				},
				Resolve: func(p graphql.ResolveParams) (interface{}, error) {
                	tag := p.Args["tag"].(string)
					song, err := getRandomSongByTag(tag)
					if err != nil {
						logrus.Errorf("failed to random song by tag, err = %v", err)
						return nil, err
					}

					return song, nil
				},
			},
		},
	})
```

Vậy query string của chúng ta sẽ thay đổi một chút thành:

```
query {
	song(tag: "us") {
    	id
  		title
		artist
    }
}
```

Kết quả sẽ random ra 1 bài hát có tag là "us":
```
{
    "song": {
        "id": 12,
        "title": "Leave out all the reset",
        "artist": "Linkin Park"
    }
}
```

## Mutation

GraphQL tư tưởng là dùng để ease for querying data là chính, nhưng với một data platform hoàn chỉnh thì phải phải hỗ trợ cả việc modify data trên server.

Bên REST không khuyến khích bạn modify data bằng `GET` request. Nhưng thật ra bạn vẫn làm được (nhét hết lên url params chẳng hạn, có điều url thì có limit length).  Bên GraphQL cũng vậy, bạn thậm chí có thể write data bằng query, nhưng nó ko đúng convention. GraphQL cung cấp `mutation` để làm chuyện này.

Ví dụ bạn muốn tạo một bài hát mới, chúng ta sẽ viết một song mutation:

```
var mutation = graphql.NewObject(graphql.ObjectConfig{
	Name: "Mutation",
	Fields: graphql.Fields{
		"createSong": &graphql.Field{
			Type:        songInputType,
			Description: "Create new song",
			Args: graphql.FieldConfigArgument{
				"title": &graphql.ArgumentConfig{
					Type: graphql.NewNonNull(graphql.String),
				},
				"artist": &graphql.ArgumentConfig{
					Type: graphql.NewNonNull(graphql.String),
				},
			},
			Resolve: func(params graphql.ResolveParams) (interface{}, error) {
				title, _ := params.Args["title"].(string)
				artist, _ := params.Args["artist"].(string)

				song := &domain.Song{
                   		Title: title,
                   		Artist: artist,
                   }

				err := saveSong(song)
				if err != nil {
                	logrus.Errorf("failed to save song, err = %v", err)
					return nil, err
				}

				return song, nil
			},
		},
	},
})

var songInputType = graphql.NewInputObject(graphql.InputObjectConfig{
	Name:        "Song",
	Description: "Song inputs",
	Fields: graphql.InputObjectConfigFieldMap{
		"title": &graphql.InputObjectFieldConfig{
			Type:        graphql.String,
			Description: "The title of the song.",
		},
		"artist": &graphql.InputObjectFieldConfig{
			Type:        graphql.String,
			Description: "The artist of the song.",
		},
	},
})

```

GraphQL phân biệt kiểu Input và Output. Các bạn có thể hiểu đơn giản là Output => cho những thứ để export ra và Input là các giá trị được truyền vào. Output và Input cùng một model nhưng có thể có schema khác nhau, Output có thể chứa nhiều complex data type hơn.

Trong ví dụ trên là songInputType sẽ là datashape của các parameters mà clients sẽ gửi lên. Khi đó mutation của chúng ta sẽ là:

```
mutation {
	createSong(title: "Co em cho", artist: "Min") {
    	id
    }
}
```
Tức là chúng ta đang muốn tạo một bài hát mới với given title + artist, sau khi tạo xong thì trả về id.
Kết quả mong đợi sẽ là

```
{
	"createSong": {
    	"id: 16
    }
}
```

Như vậy là chúng ta đã biết cách để read/write data với GraphQL.

# Combo với frontend
Đây là backend, vậy frontend thì chúng ta có gì?
- Với Vue thì bọn mình sử dụng [Vue-apolo](https://github.com/Akryum/vue-apollo)
- React thì dùng [Relay](https://github.com/facebook/relay)

Ngoài ra các bạn nào chưa muốn chuyển giao công nghệ kịp cả hai platform, thì có thể chơi thằng này [join monster](http://join-monster.readthedocs.io/en/latest/). Đại loại là một query planner, sinh ra optimal sql query. Bạn có interface cho GraphQL (viết frontend sướng quá chừng), backend thì đỡ phải cài lại query. :3.

Vã quá thì bạn nào xài posgres có thể chơi [postgraphql](https://github.com/postgraphql/postgraphql) bụp 1 cái tự instropect schema, tự gen API GraphQL luôn.

# Tổng kết

GraphQL rất thích hợp khi mà product của bạn có các clients cần flexible response format, lúc thì cần như thế này, lúc cần như thế kia mà không cần backend phải thay đổi.
Ngoài ra GraphQL cũng giúp drops TCP requests + network round trip với single endpoint.

Với GraphQL cách approach của bạn sẽ natural hơn, tức là suy nghĩ cái mình cần trả về, thay vì suy nghĩ cách lấy đầu tiên. Điều này có thể speed up development.

![alt text](https://s3-ap-southeast-1.amazonaws.com/kipalog.com/6hcpcxbkdv_image.png)

Q: GraphQL có drawbacks không?

A: Có chứ, khá nhiều đấy. Một số ví dụ như: không thể versioned, painful khi xử lý upload hay validation, cache các kiểu không safe (DataLoader)

Q: Vậy xài chung REST và GraphQL được không?

A: Hoàn toàn CÓ. Hai thằng này có thể bổ sung cho những khuyết điểm của nhau. Đôi bạn cùng tiến.



