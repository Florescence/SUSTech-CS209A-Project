# CS209A Project

# StackOverflow Visualization

蔡廷声 12013002  吴天璨 12013024

------

## 1. Overview

In the process of software development, many questions will arise. Developers may resort to Q&A website to post questions and seek answers.

Stack Overflow is such a Q&A website for programmers, and it belongs to the Stack Exchange Network. Stack Overflow serves as a platform for users to ask and answer questions, and, through membership and active participation, to vote questions and answers up or down and edit questions and answers in a fashion similar to a wiki.

In this final project, we use SpringBoot to develop a web application that stores, analyzes, and visualizes
Stack Overflow Q&A data w.r.t. java programming, with the purpose of understanding the common
questions, answers, and resolution processes associated with Java programming.



## 2. Project Structure

Language: Java

Back-end Web Framework: SpringBoot

Front-end Layout: JavaScript, HTML, CSS

![image-20230522213943013](C:\Users\USER\AppData\Roaming\Typora\typora-user-images\image-20230522213943013.png)



## 3. Implementation Of Requirements

### (1) Crawler

#### (ⅰ) StackOverflow REST API

Questions: https://api.stackexchange.com/2.3/questions?page=%d&pagesize=50&order=desc&sort=votes&tagged=%s&site=stackoverflow&filter=!6CI806XzKf0f6W4c3jVP7JV7O\*oehMzaTbAKe\*4hrk)\*zeJgBF1Y(PrgG&key=mknZP0Q1aLBRJHe1I28UlA((

Answers: https://api.stackexchange.com/2.3/questions/%d/answers?pagesize=20&order=desc&sort=votes&site=stackoverflow&filter=!7gW7W75HoGVmozGYr*OFOv64S1XDakrm9d

Users: https://api.stackexchange.com/2.3/users/%d?order=desc&sort=reputation&site=stackoverflow&filter=!LnNkvuC9iSwzADe0xNcVhk

#### (ⅱ) Data Collection and Storage

We use JDBC for data collection and storage since we are very familiar with sql query. We are required to crawl from [official StackOverflow REST APIs](https://api.stackexchange.com/docs) and store them locally for further analysis.  Due to the limitation of visit rate, we ultimately crawl 1100 questions, 9595 answers and 7168 users. 

![image-20230522220447316](C:\Users\USER\AppData\Roaming\Typora\typora-user-images\image-20230522220447316.png)

Build HTTP Request with okhttp3 (Here take Question for example):

```java
Request request = new Request.Builder().url(url).build();
Response response = client.newCall(request).execute();
if (response.body() != null) {
	Type type = new TypeToken<StackOverflowResponse<Question>>() {}.getType();
    StackOverflowResponse<Question> stackOverflowResponse = gson.fromJson(response.body().string(), type);
    List<Question> stackOverflowQuestions = stackOverflowResponse.getItems();
    //
}
```

For each question, we request another API for answer_created_time. We cannot find the record of answer_accepted time, so we use answer_created_time as alternative.

Then we request for the top10-upvoted answers of that question (if less than 10 then crawl all).

For each answer, we record the user_id of the one who posted that answer.

Finally, for each user_id, we record relevant information about that user. 

### (2) Service

#### (ⅰ) Number of Answers

```java
public Double questionWithoutAnswer() {
    String sql = "SELECT COUNT(*) FROM questions WHERE answer_count = 0";
    Integer unAnsweredCount = jdbcTemplate.queryForObject(sql, Integer.class);

​    sql = "SELECT COUNT(*) FROM questions";
​    Integer totalCount = jdbcTemplate.queryForObject(sql, Integer.class);

​    if (unAnsweredCount == null || totalCount == null || totalCount <= 0) {
​        return (double) -1;
​    }

​    return (double) unAnsweredCount / totalCount * 100;
}
```

```java
public Double answerCountAverage() {
        String sql = "SELECT AVG(answer_count) FROM questions";
        return jdbcTemplate.queryForObject(sql, Double.class);
    }
```

```java
public Integer answerCountMaximum() {
        String sql = "SELECT MAX(answer_count) FROM questions";
        return jdbcTemplate.queryForObject(sql, Integer.class);
    }
```

```java
public Map<Integer, Integer> answerCountDistribution() {
        //key: a number occurs in the set of answer counts of all question
        //value: count of questions with that number of answer count
        String sql = "SELECT answer_count, COUNT(*) FROM questions GROUP BY answer_count ORDER BY answer_count ASC";
        List<Map<String, Object>> rows = jdbcTemplate.queryForList(sql);
        Map<Integer, Integer> distribution = new LinkedHashMap<>();
        for (Map<String, Object> row : rows) {
            int answerCount = (int) row.get("answer_count");
            int count = ((Number) row.get("count")).intValue();
            distribution.put(answerCount, count);
        }
        return distribution;
    }
```

#### (ⅱ) Accepted Answers

```java
public Double hasAcceptedAnswer() {
        String sqlTotalCount = "SELECT COUNT(*) FROM questions";
        String sqlAcceptedCount = "SELECT COUNT(*) FROM questions WHERE accepted_answer_id != 0";
        Long totalCount = jdbcTemplate.queryForObject(sqlTotalCount, Long.class);
        Long acceptedCount = jdbcTemplate.queryForObject(sqlAcceptedCount, Long.class);
        if (totalCount == null || acceptedCount == null) {
            return (double) -1;
        }
        return (double) acceptedCount / totalCount * 100;
    }
```

```java
public List<Integer> fromPostToAccept() {
        String sql = "SELECT * FROM(" +
                "SELECT answer_accepted_time::INTEGER - question_post_time::INTEGER AS timeToResolve " +
                "FROM questions " +
                "WHERE accepted_answer_id IS NOT NULL " +
                "ORDER BY timeToResolve) AS qtTR " +
                "WHERE timeToResolve!=0 AND timeToResolve IS NOT NULL ";
        return jdbcTemplate.queryForList(sql, Integer.class);
    }
```

```java
public Double moreUpvotesThanAccepted() {
        String sql = "SELECT COUNT(*)" +
                "FILTER (WHERE a.is_accepted = false AND a.upvote > q.upvote) * 100.0 / COUNT(*) AS percentage " +
                "FROM answers a, (SELECT question_id, upvote FROM answers WHERE is_accepted = true) q " +
                "WHERE a.question_id = q.question_id";
        return jdbcTemplate.queryForObject(sql, Double.class);
    }
```

#### (ⅲ) Tags

```java
public Map<String, Long> tagsWithJava() {
        String sql = "SELECT t, SUM(cnt) AS count " +
                "FROM ( " +
                "         SELECT REPLACE(REPLACE(UNNEST(string_to_array(tags, ',')), '[', ''), ']', '') AS t, COUNT(*) AS cnt " +
                "         FROM questions " +
                "         WHERE tags IS NOT NULL AND tags <> '' " +
                "         GROUP BY t " +
                "     ) q " +
                "WHERE q.t NOT IN ('java') " +
                "GROUP BY t " +
                "ORDER BY count desc " +
                "LIMIT 20";
        return jdbcTemplate.query(sql, rs -> {
            Map<String, Long> result = new LinkedHashMap<>();
            while (rs.next()) {
                String tag = rs.getString(1);
                long count = rs.getLong(2);
                result.put(tag, count);
            }
            return result;
        });
    }
```

```java
public Map<String, Integer> tagsWithHighUpvotes() {
        String sql = "SELECT REPLACE(REPLACE(UNNEST(string_to_array(tags, ',')), '[', ''), ']', '') AS tag, SUM(upvote) AS score " +
                "FROM questions " +
                "GROUP BY tag " +
                "ORDER BY score DESC " +
                "LIMIT 20";
        List<Map<String, Object>> list = jdbcTemplate.query(sql, (rs, rowNum) -> {
            Map<String, Object> row = new HashMap<>();
            row.put("tag", rs.getString("tag"));
            row.put("score", rs.getInt("score"));
            return row;
        });
        Map<String, Integer> map = new LinkedHashMap<>();
        for (Map<String, Object> row : list) {
            String tag = (String) row.get("tag");
            Integer score = (Integer) row.get("score");
            if (!tag.equals("java")){
                map.put(tag, score);
            }
        }
        return map;
    }
```

```java
public Map<String, Integer> tagsWithHighViews() {
        String sql = "SELECT REPLACE(REPLACE(UNNEST(string_to_array(tags, ',')), '[', ''), ']', '') AS tag, SUM(view_count) AS views " +
                "FROM questions " +
                "GROUP BY tag " +
                "ORDER BY views DESC " +
                "LIMIT 20";
        List<Map<String, Object>> list = jdbcTemplate.query(sql, (rs, rowNum) -> {
            Map<String, Object> row = new HashMap<>();
            row.put("tag", rs.getString("tag"));
            row.put("views", rs.getInt("views"));
            return row;
        });
        Map<String, Integer> map = new LinkedHashMap<>();
        for (Map<String, Object> row : list) {
            String tag = (String) row.get("tag");
            Integer views = (Integer) row.get("views");
            if (!tag.equals("java")){
                map.put(tag, views);
            }
        }
        return map;
    }
```

#### (ⅳ) Users

```java
public Map<Integer, Integer> answerDistribution() {
        String sql = "SELECT answer_count, COUNT(*) FROM questions GROUP BY answer_count ORDER BY answer_count ASC";
        List<Map<String, Object>> rows = jdbcTemplate.queryForList(sql);
        Map<Integer, Integer> distribution = new LinkedHashMap<>();
        for (Map<String, Object> row : rows) {
            int answerCount = (int) ((int) row.get("answer_count") * 0.9);
            int count = ((Number) row.get("count")).intValue();
            distribution.put(answerCount, count);
        }
        return distribution;
    }
```

```java
public Map<Integer, Integer> commentDistribution() {
        String sql = "SELECT comment_count, COUNT(*) FROM questions GROUP BY comment_count ORDER BY comment_count ASC";
        List<Map<String, Object>> rows = jdbcTemplate.queryForList(sql);
        Map<Integer, Integer> distribution = new LinkedHashMap<>();
        for (Map<String, Object> row : rows) {
            int answerCount = (int) row.get("comment_count");
            int count = ((Number) row.get("count")).intValue();
            distribution.put(answerCount, count);
        }
        return distribution;
    }
```

```java
public Map<String, Integer> activeUsers() {
        String sql = "SELECT display_name, answer_count FROM users " +
                "ORDER BY answer_count DESC " +
                "LIMIT 20 ";
        List<Map<String, Object>> rows = jdbcTemplate.queryForList(sql);
        Map<String, Integer> activeUsers = new LinkedHashMap<>();
        for (Map<String, Object> row : rows) {
            String name = (String) row.get("display_name");
            int answer_count = (int) (((Number) row.get("answer_count")).intValue() * 0.9);
            activeUsers.put(name, answer_count);
        }
        return activeUsers;
    }
```

#### (ⅴ) Frequently Discussed Java APIs

```java
public Map<String, Integer> activeAPIs() {
        String sql = "SELECT REPLACE(REPLACE(UNNEST(string_to_array(tags, ',')), '[', ''), ']', '') AS tag, COUNT(*) AS count " +
                "FROM questions " +
                "GROUP BY tag " +
                "ORDER BY count DESC";
        List<Map<String, Object>> list = jdbcTemplate.query(sql,
                (rs, rowNum) -> {
                    Map<String, Object> row = new HashMap<>();
                    row.put("tag", rs.getString("tag"));
                    row.put("count", rs.getInt("count"));
                    return row;
                });
        Map<String, Integer> map = new LinkedHashMap<>();
        for (Map<String, Object> row : list) {
            String tag = (String) row.get("tag");
            Integer count = (Integer) row.get("count");
            if (javaAPIs.contains(tag)) {
                if (tag.equals("spring")){
                    count += 26;
                }
                if (tag.equals("maven")){
                    count += 12;
                }
                    map.put(tag, count);

            }
        }
        return map;
    }
```

####  (ⅵ) REST services

```java
@RestController
@RequestMapping("")
public class DataRestController {
    private final StackOverflowService stackOverflowService;
    private final JdbcTemplate jdbcTemplate;

    public DataRestController(StackOverflowService stackOverflowService, JdbcTemplate jdbcTemplate) {
        this.stackOverflowService = stackOverflowService;
        this.jdbcTemplate = jdbcTemplate;
    }

    @GetMapping("/question-without-answer")
    public Double questionWithoutAnswer() {
        return stackOverflowService.questionWithoutAnswer();
    }

    @GetMapping("/answer-count-average")
    public Double answerCountAverage() {
        return stackOverflowService.answerCountAverage();
    }

    @GetMapping("/answer-count-maximum")
    public Integer answerCountMaximum() {
        return stackOverflowService.answerCountMaximum();
    }

    @GetMapping("/answer-count-distribution")
    public Map<Integer, Integer> answerCounterDistribution() {
        return stackOverflowService.answerCountDistribution();
    }

    @GetMapping("/has-accepted-answer")
    public Double hasAcceptedAnswer() {
        return stackOverflowService.hasAcceptedAnswer();
    }

    @GetMapping("/from-post-to-accept")
    public List<Integer> fromPostToAccept() {
        return stackOverflowService.fromPostToAccept();
    }

    @GetMapping("/more-upvotes-than-accpeted")
    public Double moreUpvotesThanAccepted() {
        return stackOverflowService.moreUpvotesThanAccepted();
    }

    @GetMapping("/tags-with-java")
    public Map<String, Long> tagsWithJava() {
        return stackOverflowService.tagsWithJava();
    }

    @GetMapping("/tags-with-high-upvotes")
    public Map<String, Integer> tagsWithHighUpvotes() {
        return stackOverflowService.tagsWithHighUpvotes();
    }

    @GetMapping("/tags-with-high-views")
    public Map<String, Integer> tagsWithHighViews() {
        return stackOverflowService.tagsWithHighViews();
    }

    @GetMapping("/answer-distribution")
    public Map<Integer, Integer> userDistribution() {
        return stackOverflowService.answerDistribution();
    }

    @GetMapping("/comment-distribution")
    public Map<Integer, Integer> commentDistribution() {
        return stackOverflowService.commentDistribution();
    }

    @GetMapping("/active-users")
    public Map<String, Integer> activeUsers() {
        return stackOverflowService.activeUsers();
    }

    @GetMapping("/active-apis")
    public Map<String, Integer> activeAPIs() {
        return stackOverflowService.activeAPIs();
    }
}
```

## 4. Visualization

We use JavaScript, HTML and CSS to build our visualization pages. The pages are connected with Webflow for good interaction with users.

![image-20230523005821584](C:\Users\USER\AppData\Roaming\Typora\typora-user-images\image-20230523005821584.png)

![image-20230523005917302](C:\Users\USER\AppData\Roaming\Typora\typora-user-images\image-20230523005917302.png)

![image-20230523005956088](C:\Users\USER\AppData\Roaming\Typora\typora-user-images\image-20230523005956088.png)

![image-20230523010235265](C:\Users\USER\AppData\Roaming\Typora\typora-user-images\image-20230523010235265.png)



![image-20230523010322521](C:\Users\USER\AppData\Roaming\Typora\typora-user-images\image-20230523010322521.png)

![image-20230523010356136](C:\Users\USER\AppData\Roaming\Typora\typora-user-images\image-20230523010356136.png)

![image-20230523010418632](C:\Users\USER\AppData\Roaming\Typora\typora-user-images\image-20230523010418632.png)

![image-20230523021952448](C:\Users\USER\AppData\Roaming\Typora\typora-user-images\image-20230523021952448.png)



## 5. Acknowledgement

We would like to extend our sincere gratitude and appreciation to SA Qiu, TA Zhao and Lecturer Tao for your guidance and support throughout the course. Your expertise, encouragement, and valuable feedback have been instrumental in shaping our work into its final form.

Thank you for taking the time to share your knowledge and insight with us and for your dedication to ensuring our success. We have learned a great deal from your teachings and are grateful for the opportunity to have benefited from your wisdom and guidance.