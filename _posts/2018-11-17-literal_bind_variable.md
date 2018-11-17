---
layout: post
title: Literal 변수와 Bind 변수
author: sy
tags: [Literal, Bind]
---


<h2>Literal 변수</h2>

SQL문장 작성시, WHERE절의 비교되는 값에 문자/숫자 상수값을 하드코딩해서 작성한 것이다.

변수값이 빈번히 변하는 쿼리를 Literal SQL로 작성하게 되면 DB의 Library Cache 내에서 매번 다른 쿼리로 인식되어 Hard parsing을 수행한다.

운영상의 특이사항은 없지만, 시스템 성능을 저하시킬 수 있다.

자주 사용하는 Literal SQL을 확인하여, 바인드 변수를 사용하는 것을 권장한다.

* 예시<br />
SELECT * FROM EMP WHERE EMP_NO='123';

---
<h2>Bind 변수</h2>

SQL문장 작성시, WHERE절의 특정값을 표시하는 자리에 바인드 변수 형태(:B)로 작성하는 것이다.

Bind 변수를 사용하게 되면 최초 1번만 Hard parsing을 수행하고, 이후 동일 형태의 쿼리가 들어오면 이전에 수립한 실행계획(Library Cache)을 재사용한다.
불필요한 Hard parsing을 하지 않는다.

시스템 성능상에 더 효율적이다.

* 예시<br />
SELECT * FROM EMP WHERE EMP_NO=:emp_no;