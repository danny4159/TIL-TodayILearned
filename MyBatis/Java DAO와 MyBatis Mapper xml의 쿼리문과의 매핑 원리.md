# Java DAO와 MyBatis Mapper xml의 쿼리문과의 매핑 원리



아래와 같은 Mapper xml의 쿼리 문이 있다고 할 경우,

```
<insert id="insertMember">
	insert into tbl_member (userid, userpw, username, email) values 
	(#{userid}, #{userpw}, #{username}, #{email})
</insert>
```

VO 객체는 다음과 같고,

```
public class MemberVO {
	private String userid;
	private String userpw;
	private String username;
	private String email;
	private Date regdate;
	private Date updatedate;

	... 이하 getter, setter는 생략 ...
}
```

DAO에서 Mapper xml에 있는 쿼리 실행을 다음과 같이 한다고 할때,

```
public void insertMember(MemberVO vo) {
	sqlSession.insert(namespace + ".insertMember", vo);
}
```

이럴때 sqlSession.insert(namespace + ".insertMember", vo)에서 vo 객체를 Mapper로 넘기면 #{userid}, #{userpw}, #{username}, #{email} 값들과 vo가 어떻게, 어떤 원리에 의해서 매핑이 되는가?

SqlSession 클래스의 insert() 메소드를 보면 다음과 같이 API 설명이 되어 있다.
int insert(String statement, Object parameter)
  ==>Execute an insert statement with the given parameter object. Any generated autoincrement 
  ==>values or selectKey entries will modify the given parameter object properties. 

  ==>Only the number of rows affected will be returned.
  Parameters:
   -. statement : Unique identifier matching the statement to execute.
   -. parameter : A parameter object to pass to the statement.
  Returns:
   -. int : The number of rows affected by the insert.

이 메소드의 첫 번째 매개인자인 statement는 이 statement의 문자열과 1:1로 대응되는 Mapper xml 파일에 있는 쿼리문을 지칭하게 된다.
이때 Mapper의 id값과 statement의 문자열이 동일한 쿼리문에 매핑된다.
위의 예에서는 Mapper xml 파일에서 id가 insertMember인 쿼리문을 실행하게 된다.



```
<insert id="insertMember">
	insert into tbl_member (userid, userpw, username, email) values 
	(#{userid}, #{userpw}, #{username}, #{email})
</insert>
```

두 번째 매개인자인 parameter는 첫 번째 매개인자인 statement가 가리키는 Mapper 클래스의 쿼리문을 실행할 때 이 쿼리문에서 사용할 변수들에(#{userid}, #{userpw}, #{username}...) 넘겨줄 값을 담고 있다. 
위이 예에서는 MemberVO 클래스의 객체인 vo가 되겠다.
이때 vo가 클래스가 가지고 있는 값들과 #{userid}, #{userpw}, #{username}, #{email} 이 변수들이 어떤 원칙에 의해서 연동되는가 하는 것이다.

이때 두 번째 매개인자인 parameter에는 다음과 같은 다양한 종류들이 가능한데

1) parameter가 하나이고 기본 자료형이나 문자열인 경우 값이 그대로 전달된다.



```
public MemberVO readMember(String userid) throws Exception {
	return (MemberVO)sqlSession.selectOne(namespace+".selectMember", userid);
}
```

Mapper xml에서는 userid와 동일한 mapper 변수와 1:1로 대응된다. 이 경우는 #{userid} = userid가 되는 것이다.

```
<select id="selectMember" resultType="org.zerock.domain.MemberVO">
	select 
	*
	from tbl_member
	where userid = #{userid}
</select>
```

2) parameter가 클래스의 객체인 경우 해당 클래스의 getter 메소드에 대응되서 mapper 변수가 값을 획득한다.
위의 vo 객체의 경우인 #{userid}, #{userpw}, #{username}, #{email}는 각각 vo 객체의 getUserid(), getUserpw(), getUsername(), getEmail()을 통해서 이들 각각의 Mapper 변수들의 값을 할당받게 된다.
만일 MemberVO 클래스의 멤버 변수에 dayCalc는 존재하지 않지만 MemberVO 클래스에 getDayCalc()이라는 getter가 만들어져 있다면 Mapper 변수에 #{dayCalc}과 같이 표현하면 정상적으로 값을 가져올 수 있게된다.

3) parameter가 클래스의 객체는 아니나 Mapper로 넘겨야 할 파라미터가 2개 이상일 경우 Map에 담아서 넘기는데 이 경우의 1:1 대응 원칙은 다음과 같다.



```
public MemberVO readWithPW(String userid, String userpw) throws Exception {
	Map<String, Object> paramMap = new HashMap<String, Object>();
	paramMap.put("userid",  userid);
	paramMap.put("userpw", userpw);
		
	return sqlSession.selectOne(namespace+".readWithPW", paramMap);
}

<select id="readWithPW" resultType="org.zerock.domain.MemberVO">
	select
		*
	from tbl_member
	where userid = #{userid} and userpw = #{userpw}
</select>
```

이 경우는 Map 객체의 key값과 Mapper의 변수가 1:1로 대응되서 값이 전달된다.
즉 Map 객체의 key 이름과 Mapper xml의 변수 이름이 동일해야 한다.

이상의 원리를 따라 MyBatis Mapper xml의 쿼리문과의 연동은 SqlSession 클래스가 알아서 매핑 작업을 처리해 준다.



출처: https://developer-joe.tistory.com/235 [코드 조각-Android, Java, Spring, JavaScript, C#, C, C++, PHP, HTML, CSS, Delphi]