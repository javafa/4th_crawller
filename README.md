# 4th_crawller
```javascript
// http 통신으로 request를 할 모듈
var request = require("request")
var querystring = require("querystring")
var cheerio = require("cheerio")
// mysql db 연결정보
var mysql = require("mysql")
var conInfo = {
    host : '127.0.0.1',
    user: 'root',
    password : 'mysql',
    port : '3306',
    database : 'socar'
}
var con = mysql.createConnection(conInfo);
con.connect();
var query_str = "select * from crawl_zone"

var result;
var result_idx = 0;

// zone 테이블 select
con.query(query_str, function(error, items, fields){
    result = items;
    runRequest();
});

function runRequest(){
    // 테이블에서 한개의 row를 꺼내고
    var obj = result[result_idx];
    // 웹사이트로 request 를 다시 전송한다.
    var form = {
        way: "round",
        start_at: "2017-11-21 15:00",
        end_at: "2017-11-21 15:30",
        loc_type: 2,
        class_id: "",
        lat: obj.lat,
        lng: obj.lng,
        zone_id: obj.zone_id,
        region_name: obj.zone_name,
        distance: 1
    };

    // form 데이터를 문자열로 변환
    var formData = querystring.stringify(form);
    var formLength = formData.length;

    request(
        {
            headers : {
                'Content-Length' : formLength,
                'Content-Type': 'application/x-www-form-urlencoded'
            },
            url : "https://www.socar.kr/reserve/search",
            body : formData,
            method : "POST"
        },
        function(error, answer, body){
            if(error){
                console.log(error)
            }else{
                if(body){ // html 데이터가 반환된다.
                    var start = body.indexOf('<!-- list -->');
                    var end = body.indexOf('<!-- //list -->');
                    body = body.substring(start,end);
                    
                    var sections = body.split('<div class="section">');
                    
                    for(var i=1; i<sections.length; i++){
                        var section = sections[i];
                        var html = cheerio.load(section);
                        var zone_id = html(".arti > em").text();
                        var zone_name = html(".arti > h4").text();
                        var car_name = html(".carInfo .desc > h5").text();
                        var oil_type = html(".carInfo .desc .spec > em").text();
                        var option = html(".carInfo .desc .spec").text();
                        option = option.split("옵션 : ");
                        option = option[1].split("\n")[0];
                        var price = html(".price.price_info #price-s0").text();
                        var driving_fee = html(".oil").text();

                        // console.log("zone_id:"+zone_id);
                        // console.log("zone_name:"+zone_name);
                        // console.log("car name="+car_name);
                        // console.log("oil_type="+oil_type);
                        // console.log("option="+option);
                        // console.log("price="+price);
                        // console.log("driving_fee="+driving_fee);

                        query_str = "insert into crawl_car(zone_id,zone_name";
                        query_str += ",car_name,oil_type,car_option,member_discount";
                        query_str += ",driving_fee) values(?,?,?,?,?,?,?)"
                        values = [zone_id,zone_name,car_name,oil_type,option,price,driving_fee];
                        con.query(query_str, values, function(error,result){
                            if(error){
                                console.log(error);
                            }else{
                                console.log("insertion completed! : "+ zone_id);
                            }
                        });
                    }
                    
                    //console.log(body);
                }
            }
        }
    );

    result_idx++
    if(result_idx < result.length)
        setTimeout(runRequest,1000);
}


```
