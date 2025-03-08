# QueryDSL 페이징

 - 출처
    - https://hstory0208.tistory.com/entry/QueryDSL-%ED%8E%98%EC%9D%B4%EC%A7%95-%EC%B2%98%EB%A6%AC

## 참고사항

### fetchResults() deprecated 

기존 QueryDSL에서 fetchResults() 메서드를 제공하여 해당 메서드를 사용해 조회 데이터와 카운트 쿼리 데이터를 한 번에 조회할 수 있었다.

하지만, 성능 문제가 있었는지 현재는 deprecated 되어, 조회 데이터와 카운트 쿼리를 개발자가 직접 호출해야 한다.

### PageableExecutionUtils

PageableExecutionUtils를 사용해 반환하면 content(조회 데이터)와 pageable의 total size를 확인해 성능 최적화가 발생한다.

 - 페이지의 시작이면서 컨텐츠 사이즈가 pageSize보다 작거나 마지막 페이지일 때 카운트 쿼리를 실행하지 않는다.

### 정렬(ORDER BY)

QueryDSL에서는 Spring Data JPA의 Sort를 사용할 수 없다.

정렬 기능을 사용하기 위해서는 QueryDSL의 OrderSpecifier 클래스를 직접 정의해주어야 한다.
 - PathBuilder를 사용해 동적으로 정렬할 필드를 지정
 - Order.ASC, Order.DESC를 이용해 정렬 방향 설정
```java
import com.querydsl.core.types.Order;
import com.querydsl.core.types.OrderSpecifier;
import com.querydsl.core.types.dsl.PathBuilder;

// 정렬 단건
private OrderSpecifier<?> getSort(String sortBy, Sort.Direction sortDirection) {
    PathBuilder<?> entityPath = new PathBuilder<>(Member.class, "member");
    Order order = sortDirection.isAscending() ? Order.ASC : Order.DESC;
    return new OrderSpecifier<>(order, entityPath.get(sortBy));
}

OrderSpecifier<?> orderSpecifier = getSort(filter.getSortBy(), filter.getSortDirection());

query.select(member)
    .from(member)
    .orderBy(orderSpecifier)
    .fetch();


// 정렬 여러건
private List<OrderSpecifier<?>> getSort(List<String> sortByList, List<Sort.Direction> sortDirectionList) {
    List<OrderSpecifier<?>> orderSpecifiers = new ArrayList<>();
    PathBuilder<?> entityPath = new PathBuilder<>(Member.class, "member");

    for (int i = 0; i < sortByList.size(); i++) {
        String sortBy = sortByList.get(i);
        Sort.Direction direction = (i < sortDirectionList.size()) ? sortDirectionList.get(i) : Sort.Direction.DESC;

        Order order = direction.isAscending() ? Order.ASC : Order.DESC;
        orderSpecifiers.add(new OrderSpecifier<>(order, entityPath.get(sortBy)));
    }

    return orderSpecifiers;
}


List<OrderSpecifier<?>> orderSpecifiers = getSort(filter.getSortBy(), filter.getSortDirection());

query.select(member)
    .from(member)
    .orderBy(orderSpecifiers.toArray(new OrderSpecifier[0]))
    .fetch();
```

## 페이징 예시

 - `Controller`
```java
@GetMapping("/members")
public Page<MemberDto> findMemberList(@ModelAttribute MemberFindRequest request, Pageable pageable) {
    MemberFindCriteria criteria = request.toMemberFindCriteria(pageable);
    return memberRepository.findMemberList(findMemberList);
}
```

 - `Repository`
```java
public interface MemberRepository {
    Page<MemberDto> findMemberList(MemberFindCriteria);
}

@Repository
public class MemberRepositoryImpl implements MemberRepository {

    private final JPAQueryFactory factory;

    @Override
    public Page<MemberDto> findMemberList(MemberFindCriteria criteria) {
        Pageable pageable = criteria.getPageable();

        List<MemberDto> list = factory
            .select(memberEntity)
            .from(memberEntity)
            .where(nameEq(criteria.getName()),
                    ageGoe(criteria.getAgeGoe()),
                    ageLoe(criteria.getAgeLoe()))
            .offset(pageable.getOffset())   // 페이지 시작 위치
            .limit(pageable.getPageSize())  // 페이지당 행 갯수
            .fetch();
        
        JPAQuery<Long> countQuery = factory
            .select(member.count())
            .from(member)
            .where(nameEq(criteria.getName()),
                    ageGoe(criteria.getAgeGoe()),
                    ageLoe(criteria.getAgeLoe()));
        
        return PageableExecutionUtils.getPage(list, pageable, countQuery::fetchOne);
    }

    private BooleanExpression nameEq(String name) {
        return hasText(name) ? member.name.eq(name) : null;
    }

    private BooleanExpression ageGoe(Integer ageGoe) {
        return ageGoe == null ? null : member.age.goe(ageGoe);
    }

    private BooleanExpression ageLoe(Integer ageLoe) {
        return ageLoe == null ? null : member.age.loe(ageLoe);
    }
}
```
