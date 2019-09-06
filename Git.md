# 1. Check xem branch đã được merge chưa
Source: https://stackoverflow.com/questions/226976/how-can-i-know-if-a-branch-has-been-already-merged-into-master

Sử dụng lệnh `git merge-base` để tìm commit chung gần nhất giữa 2 branch.

```
git merge-base <commit> <commit>
```

Dùng tham số `--is-ancestor` để check xem HEAD của branch 1 có là cha của HEAD của branch 2 không:

```
git merge-base --is-ancestor <branch1> <branch2>
```

Ví dụ:

```
git merge-base --is-ancestor feature/new-feature develop
```

Nếu câu lệnh chạy thành công (mã lỗi 0) thì branch `feature/new-feature` đã được merge vào `develop`.
Còn nếu chạy ra mã lỗi thì tức là chưa được merge, hoặc đã từng merge rồi nhưng xuất hiện thêm commit mới.

Có thể dùng function sau để check:

```sh
git-is-merged () {
  merge_destination_branch=$1
  merge_source_branch=$2

  merge_base=$(git merge-base $merge_destination_branch $merge_source_branch)
  merge_source_current_commit=$(git rev-parse $merge_source_branch)
  if [[ $merge_base = $merge_source_current_commit ]]
  then
    echo $merge_source_branch is merged into $merge_destination_branch
    return 0
  else
    echo $merge_source_branch is not merged into $merge_destination_branch
    return 1
  fi
}
```

```sh
git-is-merged develop feature/new-feature
```