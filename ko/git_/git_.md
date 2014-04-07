# Git 레퍼런스

`git log 1a410e` 라고 실행하면 전체 히스토리를 볼 수 있지만, 여전히 `1a410e`를 기억해야 한다. 이 커밋은 마지막 커밋이기 때문에 히스토리를 따라 모든 개체를 조회할 수 있다. SHA-1 값을 날로 사용하기보다 쉬운 이름으로 된 포인터가 있으면 그걸 사용하는게 더 좋다. 외우기 쉬운 이름으로 된 파일에 SHA-1 값을 저장한다.

Git에서는 이런 것을 "레퍼런스" 또는 "refs"라고 부른다. SHA-1 값이 든 파일은 `.git/refs` 디렉토리에 있다. 이 프로젝트에는 아직 레퍼런스가 하나도 없다. 이 디렉토리의 구조는 매우 단순하다:

	$ find .git/refs
	.git/refs
	.git/refs/heads
	.git/refs/tags
	$ find .git/refs -type f
	$

레퍼런스가 있으면 마지막 커밋이 무엇인지 기억하기 쉽다. 사실 내부는 아래처럼 단순하다:

	$ echo "1a410efbd13591db07496601ebc7a059dd55cfe9" > .git/refs/heads/master

SHA-1 값 대신에 지금 만든 레퍼런스를 사용할 수 있다:

	$ git log --pretty=oneline  master
	1a410efbd13591db07496601ebc7a059dd55cfe9 third commit
	cac0cab538b970a37ea1e769cbbde608743bc96d second commit
	fdf4fc3344e67ab068f836878b6c4951e3b15f3d first commit

레퍼런스 파일을 직접 고치는 것이 좀 못마땅하다. Git에는 좀 더 안전하게 바꿀 수 있는 `update-ref` 명령이 있다:

	$ git update-ref refs/heads/master 1a410efbd13591db07496601ebc7a059dd55cfe9

Git 브랜치의 역할이 바로 이거다. 브랜치는 어떤 작업들 중 마지막 작업을 가리키는 포인터 또는 레퍼런스이다. 간단히 두 번째 커밋을 가리키는 브랜치를 만들어 보자:

	$ git update-ref refs/heads/test cac0ca

브랜치는 직접 가리키는 커밋과 그 커밋으로 따라갈 수 있는 모든 커밋을 포함한다:

	$ git log --pretty=oneline test
	cac0cab538b970a37ea1e769cbbde608743bc96d second commit
	fdf4fc3344e67ab068f836878b6c4951e3b15f3d first commit

이제 Git 데이터베이스는 그림 9-4처럼 보인다.


![](http://git-scm.com/figures/18333fig0904-tn.png)

그림 9-4. 브랜치 레퍼런스가 추가된 Git 데이터베이스

`git branch (branchname)` 명령을 실행하면 Git은 내부적으로 `update-ref` 명령을 실행한다. 입력받은 브랜치 이름과 현 브랜치의 마지막 커밋의 SHA-1 값을 가져다 `update-ref` 명령을 실행한다.

## HEAD

`git branch (branchname)` 명령을 실행할 때 Git은 어떻게 마지막 커밋의 SHA-1 값을 아는 걸까? HEAD 파일은 현 브랜치를 가리키는 간접(symbolic) 레퍼런스다. 간접 레퍼런스이기 때문에 다른 레퍼런스와 다르다. 이 레퍼런스은 다른 레퍼런스를 가리키는 것이라서 SHA-1 값이 없다. 파일을 열어 보면 아래와 같이 생겼다:

	$ cat .git/HEAD
	ref: refs/heads/master

`git checkout test`를 실행하면 Git은 HEAD 파일을 아래와 같이 바꾼다:

	$ cat .git/HEAD
	ref: refs/heads/test

`git commit`을 실행하면 커밋 개체가 만들어지는데, 지금 HEAD가 가리키고 있던 커밋의 SHA-1 값이 그 커밋 개체의 부모로 사용된다.

이 파일도 손으로 직접 편집할 수 있지만  `symbolic-ref`라는 명령어가 있어서 좀 더 안전하게 사용할 수 있다. 이 명령으로 HEAD의 값을 읽을 수 있다:

	$ git symbolic-ref HEAD
	refs/heads/master

HEAD의 값을 변경할 수도 있다:

	$ git symbolic-ref HEAD refs/heads/test
	$ cat .git/HEAD
	ref: refs/heads/test

refs 형식에 맞지 않으면 수정할 수 없다:

	$ git symbolic-ref HEAD test
	fatal: Refusing to point HEAD outside of refs/

## 태그

중요한 개체는 모두 살펴봤고 남은 개체가 하나 있다. 태그 개체는 커밋 개체랑 매우 비슷하다. 커밋 개체처럼 누가, 언제 태그를 달았는지 태그 메시지는 무엇이고 어떤 커밋을 가리키는지에 대한 정보가 포함된다. 태그 개체는 Tree 개체가 아니라 커밋 개체를 가리키는 것이 그 둘의 차이다. 브랜치처럼 커밋 개체를 가리키지만 옮길 수는 없다. 태그 개체는 늘 그 이름이 뜻하는 커밋만 가리킨다.

*2장*에서 배웠듯이 태그는 Annotated 태그와 Lightweight 태그 두 종류로 나뉜다. 먼저 아래와 같이 Lightweight 태그를 만들어 보자:

	$ git update-ref refs/tags/v1.0 cac0cab538b970a37ea1e769cbbde608743bc96d

Lightwieght 태그는 만들기 쉽다. 브랜치랑 비슷하지만 브랜치처럼 옮길 수는 없다. 이에 비해 Annotated 태그는 좀 더 복잡하다. Annotated 태그를 만들면 Git은 태그 개체를 만들고 거기에 커밋을 가리키는 레퍼런스를 저장한다. Annotated 태그는 커밋을 직접 가리키지 않고 태그 개체를 가리킨다. `-a` 옵션을 주고 Annotated 태그를 만들고 확인해보자:

	$ git tag -a v1.1 1a410efbd13591db07496601ebc7a059dd55cfe9 -m 'test tag'

태그 개체의 SHA-1 값을 확인한다:

	$ cat .git/refs/tags/v1.1
	9585191f37f7b0fb9444f35a9bf50de191beadc2

`cat-file` 명령으로 해당 SHA-1 값의 내용을 조회한다:

	$ git cat-file -p 9585191f37f7b0fb9444f35a9bf50de191beadc2
	object 1a410efbd13591db07496601ebc7a059dd55cfe9
	type commit
	tag v1.1
	tagger Scott Chacon <schacon@gmail.com> Sat May 23 16:48:58 2009 -0700

	test tag

`object` 부분에 있는 SHA-1 값이 실제로 태그가 가리키는 커밋이다. 커밋 개체뿐만 아니라 모든 Git 개체에 태그를 달 수 있다. 커밋 개체에 태그를 다는 것이 아니라 Git 개체에 태그를 다는 것이다. Git을 개발하는 프로젝트에서는 관리자가 자신의 GPG 공개키를 Blob 개체로 추가하고 그 파일에 태그를 달았다. 다음 명령으로 그 공개키를 확인할 수 있다:

	$ git cat-file blob junio-gpg-pub

Linux Kernel 저장소에도 커밋이 아닌 다른 개체를 가리키는 태그 개체가 있다. 그 태그는 저장소에 처음으로 소스 코드를 임포트했을 때 그 첫 Tree 개체를 가리킨다.

## 리모트 레퍼런스

리모트 레퍼런스라는 것도 있다. 리모트를 추가하고 Push하면 Git은 각 브랜치마다 Push한 마지막 커밋이 무엇인지 `refs/remotes` 디렉토리에 저장한다. 예를 들어, `origin`이라는 리모트를 추가하고 `master` 브랜치를 Push 한다:

	$ git remote add origin git@github.com:schacon/simplegit-progit.git
	$ git push origin master
	Counting objects: 11, done.
	Compressing objects: 100% (5/5), done.
	Writing objects: 100% (7/7), 716 bytes, done.
	Total 7 (delta 2), reused 4 (delta 1)
	To git@github.com:schacon/simplegit-progit.git
	   a11bef0..ca82a6d  master -> master

`origin`의 `master` 브랜치에서 서버와 마지막으로 교환한 커밋이 어떤 것인지 `refs/remotes/origin/master` 파일에서 확인할 수 있다:

	$ cat .git/refs/remotes/origin/master
	ca82a6dff817ec66f44342007202690a93763949

리모트 레퍼런스는 Checkout할 수 없다. `refs/heads`에 있는 레퍼런스인 브랜치와 다르다. 이 리모트 레퍼런스는 서버의 브랜치가 가리키는 커밋이 무엇인지 적어둔 일종의 북마크이다.
. 그리고 나서 Tree 개체를 만든다:

	$ echo 'new file' > new.txt
	$ git update-index test.txt
	$ git update-index --add new.txt

새 파일인 new.txt와 새로운 버전의 test.txt 파일까지 Staging Area에 추가했다. 현재 상태의 Staging Area를 새로운 Tree 개체로 기록하면 어떻게 보이는지 살펴보자:

	$ git write-tree
	0155eb4229851634a0f03eb265b69f5a2d56f341
	$ git cat-file -p 0155eb4229851634a0f03eb265b69f5a2d56f341
	100644 blob fa49b077972391ad58037050f2a75f74e3671e92      new.txt
	100644 blob 1f7a7a472abf3dd9643fd615f6da379c4acb3e3a      test.txt

이 Tree 개체에는 파일이 두 개 있고 test.txt 파일의 SHA 값도 두 번째 버전인 `1f7a7a1`이다. 재미난 걸 해보자. 처음에 만든 Tree 개체를 하위 디렉토리로 만들 수 있다. `read-tree` 명령으로 Tree 개체를 읽어 Staging Area에 추가한다. `--prefix` 옵션을 주면 Tree 개체를 하위 디렉토리로 추가할 수 있다.

	$ git read-tree --prefix=bak d8329fc1cc938780ffdd9f94e0d364e0ea74f579
	$ git write-tree
	3c4e9cd789d88d8d89c1073707c3585e41b0e614
	$ git cat-file -p 3c4e9cd789d88d8d89c1073707c3585e41b0e614
	040000 tree d8329fc1cc938780ffdd9f94e0d364e0ea74f579      bak
	100644 blob fa49b077972391ad58037050f2a75f74e3671e92      new.txt
	100644 blob 1f7a7a472abf3dd9643fd615f6da379c4acb3e3a      test.txt

이 Tree 개체로 워킹 디렉토리를 만들면 파일 두 개와 `bak`이라는 하위 디렉토리가 생긴다. 그리고 `bak` 디렉토리 안에는 test.txt 파일의 처음 버전이 들어 있다. 그림 9-2와 같은 구조로 데이터가 저장된다.


![](http://git-scm.com/figures/18333fig0902-tn.png)

그림 9-2. 현재 Git 데이터 구조

## 커밋 개체

각기 다른 스냅샷을 나타내는 Tree 개체를 세 개 만들었다. 하지만, 여전히 이 스냅샷을 불러 오려면 SHA-1 값을 기억하고 있어야 한다. 스냅샷을 누가, 언제, 왜 저장했는지에 대한 정보는 아예 없다. 이런 정보는 커밋 개체에 저장된다:

커밋 개체는 `commit-tree` 명령으로 만든다. 이 명령에 커밋 개체에 대한 설명과 Tree 개체의 SHA-1 값 한 개를 넘긴다. 앞서 저장한 첫 번째 Tree를 가지고 아래와 같이 만들어 본다:

	$ echo 'first commit' | git commit-tree d8329f
	fdf4fc3344e67ab068f836878b6c4951e3b15f3d

새로 생긴 커밋 개체를 `cat-file` 명령으로 확인해보자:

	$ git cat-file -p fdf4fc3
	tree d8329fc1cc938780ffdd9f94e0d364e0ea74f579
	author Scott Chacon <schacon@gmail.com> 1243040974 -0700
	committer Scott Chacon <schacon@gmail.com> 1243040974 -0700

	first commit

커밋 개체의 형식은 간단하다. 해당 스냅샷에서 최상단 Tree를(역주 - 루트 디렉터리 같은) 하나 가리킨다. 그리고 `user.name`과 `user.email` 설정에서 가져온 Author/Committer 정보, 시간 정보, 그리고 한 줄 띄운 다음 커밋 메시지가 들어 간다.

이제 커밋 개체를 두 개 더 만들어 보자. 각 커밋 개체는 이전 개체를 가리키도록 한다:

	$ echo 'second commit' | git commit-tree 0155eb -p fdf4fc3
	cac0cab538b970a37ea1e769cbbde608743bc96d
	$ echo 'third commit'  | git commit-tree 3c4e9c -p cac0cab
	1a410efbd13591db07496601ebc7a059dd55cfe9

세 커밋 개체는 각각 해당 스냅샷을 나타내는 Tree 개체를 하나씩 가리키고 있다. 이상해 보이겠지만 우리는 진짜 Git 히스토리를 만들었다. 마지막 커밋 개체의 SHA-1 값을 주고 `git log` 명령을 실행하면 아래와 같이 출력한다:

	$ git log --stat 1a410e
	commit 1a410efbd13591db07496601ebc7a059dd55cfe9
	Author: Scott Chacon <schacon@gmail.com>
	Date:   Fri May 22 18:15:24 2009 -0700

	    third commit

	 bak/test.txt |    1 +
	 1 files changed, 1 insertions(+), 0 deletions(-)

	commit cac0cab538b970a37ea1e769cbbde608743bc96d
	Author: Scott Chacon <schacon@gmail.com>
	Date:   Fri May 22 18:14:29 2009 -0700

	    second commit

	 new.txt  |    1 +
	 test.txt |    2 +-
	 2 files changed, 2 insertions(+), 1 deletions(-)

	commit fdf4fc3344e67ab068f836878b6c4951e3b15f3d
	Author: Scott Chacon <schacon@gmail.com>
	Date:   Fri May 22 18:09:34 2009 -0700

	    first commit

	 test.txt |    1 +
	 1 files changed, 1 insertions(+), 0 deletions(-)

놀랍지 않은가! 방금 우리는 고수준 명령어 없이 저수준의 명령으로만 Git 히스토리를 만들었다. 지금 한 일이 `git add`와 `git commit` 명령을 실행했을 때 Git 내부에서 일어나는 일이다. Git은 변경된 파일을 Blob 개체로 저장하고 현 Index에 따라서 Tree 개체를 만든다. 그리고 이전 커밋 개체와 최상위 Tree 개체를 참고해서 커밋 개체를 만든다. 즉 Blob, Tree, 커밋 개체가 Git의 주요 개체이고 이 개체는 전부 `.git/objects` 디렉토리에 저장된다. 이 예제에서 생성한 개체는 아래와 같다:

	$ find .git/objects -type f
	.git/objects/01/55eb4229851634a0f03eb265b69f5a2d56f341 # tree 2
	.git/objects/1a/410efbd13591db07496601ebc7a059dd55cfe9 # commit 3
	.git/objects/1f/7a7a472abf3dd9643fd615f6da379c4acb3e3a # test.txt v2
	.git/objects/3c/4e9cd789d88d8d89c1073707c3585e41b0e614 # tree 3
	.git/objects/83/baae61804e65cc73a7201a7252750c76066a30 # test.txt v1
	.git/objects/ca/c0cab538b970a37ea1e769cbbde608743bc96d # commit 2
	.git/objects/d6/70460b4b4aece5915caf5c68d12f560a9fe3e4 # 'test content'
	.git/objects/d8/329fc1cc938780ffdd9f94e0d364e0ea74f579 # tree 1
	.git/objects/fa/49b077972391ad58037050f2a75f74e3671e92 # new.txt
	.git/objects/fd/f4fc3344e67ab068f836878b6c4951e3b15f3d # commit 1

내부의 포인터를 따라가면 그림 9-3과 같은 그래프가 그려진다.


![](http://git-scm.com/figures/18333fig0903-tn.png)

그림 9-3. Git 저장소 내의 모든 개체

## 개체 저장소

내용과 함께 헤더도 저장한다고 얘기했다. 잠시 Git이 개체를 어떻게 저장하는지부터 살펴보자. "what is up, doc?"이라는 문자열을 가지고 대화형 Ruby 쉘 `irb` 명령어로 흉내 내 보자:

	$ irb
	>> content = "what is up, doc?"
	=> "what is up, doc?"

Git은 개체의 타입을 시작으로 헤더를 만든다. 그다음에 공백 문자 하나, 내용의 크기, 마지막에 널 문자를 추가한다:

	>> header = "blob #{content.length}\0"
	=> "blob 16\000"

Git은 헤더와 원래 내용을 합쳐서 SHA-1 체크섬을 계산한다. Ruby에서도 `require`로 SHA1 라이브러리를 가져다가 흉내 낼 수 있다. `require`로 라이브러리를 포함하고 나서 `Digest::SHA1.hexdigest()`를 호출한다:

	>> store = header + content
	=> "blob 16\000what is up, doc?"
	>> require 'digest/sha1'
	=> true
	>> sha1 = Digest::SHA1.hexdigest(store)
	=> "bd9dbf5aae1a3862dd1526723246b20206e5fc37"

Git은 또 zlib으로 내용을 압축한다. Ruby에도 zlib 라이브러리가 있으니 Ruby에서도 할 수 있다. 라이브러리를 포함하고 `Zlib::Deflate.deflate()`를 호출한다:

	>> require 'zlib'
	=> true
	>> zlib_content = Zlib::Deflate.deflate(store)
	=> "x\234K\312\311OR04c(\317H,Q\310,V(-\320QH\311O\266\a\000_\034\a\235"

마지막으로 zlib으로 압축한 내용을 개체로 저장한다. SHA-1 값 중에서 맨 앞에 있는 두 자를 가져다 하위 디렉토리 이름으로 사용하고 나머지 38자를 그 디렉토리 안에 있는 파일이름으로 사용한다. Ruby에서는 `FileUtils.mkdir_p()`로 하위 디렉토리의 존재를 보장하고 나서 `File.open()`으로 파일을 연다. 그리고 그 파일에 zlib으로 압축한 내용을 `write()` 함수로 저장한다.

	>> path = '.git/objects/' + sha1[0,2] + '/' + sha1[2,38]
	=> ".git/objects/bd/9dbf5aae1a3862dd1526723246b20206e5fc37"
	>> require 'fileutils'
	=> true
	>> FileUtils.mkdir_p(File.dirname(path))
	=> ".git/objects/bd"
	>> File.open(path, 'w') { |f| f.write zlib_content }
	=> 32

다 됐다. 이제 Git Blob 개체를 손으로 만들었다. Git 개체는 모두 이 방식으로 저장하며 단지 종류만 다르다. 헤더가 `blob`이 아니라 그냥 `commit`이나 `tree`로 시작하게 되는 것뿐이다. Blob 개체는 여기서 보여준 것과 거의 같지만 커밋이 개체나 Tree 개체는 각기 다른 형식을 사용한다.
�고 그 프로젝트 디렉토리로 이동해서 이 명령의 표준출력을 `git fast-import` 명령의 표준입력으로 연결한다(pipe).

	$ git init
	Initialized empty Git repository in /opt/import_to/.git/
	$ ruby import.rb /opt/import_from | git fast-import
	git-fast-import statistics:
	---------------------------------------------------------------------
	Alloc'd objects:       5000
	Total objects:           18 (         1 duplicates                  )
	      blobs  :            7 (         1 duplicates          0 deltas)
	      trees  :            6 (         0 duplicates          1 deltas)
	      commits:            5 (         0 duplicates          0 deltas)
	      tags   :            0 (         0 duplicates          0 deltas)
	Total branches:           1 (         1 loads     )
	      marks:           1024 (         5 unique    )
	      atoms:              3
	Memory total:          2255 KiB
	       pools:          2098 KiB
	     objects:           156 KiB
	---------------------------------------------------------------------
	pack_report: getpagesize()            =       4096
	pack_report: core.packedGitWindowSize =   33554432
	pack_report: core.packedGitLimit      =  268435456
	pack_report: pack_used_ctr            =          9
	pack_report: pack_mmap_calls          =          5
	pack_report: pack_open_windows        =          1 /          1
	pack_report: pack_mapped              =       1356 /       1356
	---------------------------------------------------------------------

성공적으로 끝나면 여기서 보여주는 것처럼 어떻게 됐는지 통계를 보여준다. 이 경우엔 브랜치 1개와 커밋 5개 그리고 개체 18개가 임포트됐다. `git log` 명령으로 히스토리 조회가 가능하다:

	$ git log -2
	commit 10bfe7d22ce15ee25b60a824c8982157ca593d41
	Author: Scott Chacon <schacon@example.com>
	Date:   Sun May 3 12:57:39 2009 -0700

	    imported from current

	commit 7e519590de754d079dd73b44d695a42c9d2df452
	Author: Scott Chacon <schacon@example.com>
	Date:   Tue Feb 3 01:00:00 2009 -0700

	    imported from back_2009_02_03

이 시점에서는 아무것도 Checkout하지 않았기 때문에 워킹 디렉토리에 아직 아무 파일도 없다. `master` 브랜치로 Reset해서 파일을 Checkout한다:

	$ ls
	$ git reset --hard master
	HEAD is now at 10bfe7d imported from current
	$ ls
	file.rb  lib

`fast-import` 명령으로 많은 일을 할 수 있다. 모드를 설정하고, 바이너리 데이터를 다루고, 브랜치를 여러 개 다루고, Merge하고, 태그를 달고, 진행상황을 보여 주고, 등등 무수히 많은 일을 할 수 있다. Git 소스의 `contrib/fast-import` 디렉토리에 복잡한 상황을 다루는 예제가 많다. 그 중 여기서 설명한 `git-p4` 스크립트가 좋은 예제이다.
