# Chapter 02 Mission

### Diagram
![diagram.png](imgs%2Fdiagram.png)

### Mission

1. 내가 진행중인 미션, 진행 완료한 미션 (페이징포함)
```sql
# parameter = userId, missionState(진행여부), offset

SELECT 
	*
FROM 
	user_mission um
LEFT JOIN 
	mission m ON um.mission_id = m.id
LEFT JOIN
	store s ON um.store_id = s.id
WHERE 
	um.user_id = ?(userId)
	AND
	um.state = ?(missionState)
ORDER BY 
	um.created_at DESC
LIMIT
	10 
OFFSET 
	(?(offset)-1)*10
```


2. 리뷰 작성하는 쿼리
```sql
# parameter = userId, storeId, title, content, score

INSERT INTO 
	review(user_id, store_id, title, content, score)
VALUE
	(?, ?, ?, ?, ?)
```

3. 홈 화면 
- 현재 달성한 미션의 개수 % 10
- 진행하지 않은 미션

```sql
# parameter = userId, location, offset

# 현재 달성한 미션의 개수 / 10
SELECT 
	count(*)%10
FROM 
	user_mission um
INNER JOIN
	mission m ON um.mission_id = m.id
INNER JOIN
	store s ON m.store_id = s.id
WHERE
	um.user_id = ?(userId)
	AND
	s.location = ?(location)
	AND
	um.state = 'COMPLETE'
```

```sql
# 진행하지 않은 미션
select 
    m.id,
	s.store_name,
	sc.store_category_name,
	m.content,
	m.point,
	DATEDIFF(m.due_date, current_timestamp)
FROM
	mission m
LEFT JOIN
	(SELECT * FROM user_mission WHERE user_mission.user_id = ?(userId)) um ON m.id = um.mission_id
LEFT JOIN
	store s ON m.store_id = s.id
LEFT JOIN
	store_category sc ON s.category_id = sc.id
WHERE
	um.user_id IS NULL
ORDER BY
	m.created_at DESC
LIMIT
	10 
OFFSET
	(?(offset)-1) * 10
```

진행하지 않은 미션의 경우, 현재 사용자가 진행중, 완료한 미션을 제외한 나머지 미션을 가져와야하므로 
LEFT JOIN을 사용하여 mission 테이블과 user_mission 테이블을 조인하고, user_mission 
테이블의 user_id가 NULL인 경우만 가져오도록 합니다.

해당 방법 이외에 NOT IN 을 사용할 수도 있지만, 미션의 개수가 많을 수록 비교의 횟수가 매우 증가하므로 
쿼리의 성능이 훨씬 떨어질 수 있습니다.

4. 마이페이지
```sql
SELECT 
	user.email,
	user.name,
	user.point,
	CASE
		WHEN (user.phone_auth = 0) THEN '미인증'
		ELSE user.phone_num
	END AS 휴대폰인증
FROM 
	user
where
	user.id = ? (userId)
```