# Git 物件

Git 是一套內容定址檔案系統。很不錯。不過這是什麼意思呢？
這種說法的意思是，從內部來看，Git 是簡單的 key-value 資料儲存。它允許插入任意類型的內容，並會回傳一個鍵值，通過該鍵值可以在任何時候再取出該內容。可以通過底層命令 `hash-object` 來示範這點，傳一些資料給該命令，它會將資料保存在 `.git` 目錄並回傳表示這些資料的鍵值。首先初使化一個 Git 倉庫並確認 `objects` 目錄是空的： 

	$ mkdir test
	$ cd test
	$ git init
	Initialized empty Git repository in /tmp/test/.git/
	$ find .git/objects
	.git/objects
	.git/objects/info
	.git/objects/pack
	$ find .git/objects -type f
	$

Git 初始化了 `objects 目錄`，同時在該目錄下創建了 `pack` 和 `info` 子目錄，但是該目錄下沒有其他常規檔。我們往這個 Git 資料庫裡儲存一些文本：

	$ echo 'test content' | git hash-object -w --stdin
	d670460b4b4aece5915caf5c68d12f560a9fe3e4

參數 `-w` 指示 `hash-object` 命令儲存 (資料) 物件，若不指定這個參數該命令僅僅回傳鍵值。`--stdin` 指定從標準輸入裝置 (stdin) 來讀取內容，若不指定這個參數，`hash-object` 就需要指定一個要儲存的檔案路徑。該命令輸出長度為 40 個字元的校驗和(checksum hash)。這是個 SHA-1 雜湊值──其值為要儲存的資料加上你馬上會瞭解到的一種頭資訊(header)的校驗和。現在可以查看到 Git 已經儲存了資料： 

	$ find .git/objects -type f
	.git/objects/d6/70460b4b4aece5915caf5c68d12f560a9fe3e4

可以在 `objects` 目錄下看到一個檔。這便是 Git 儲存資料內容的方式──為每份內容生成一個檔，取得該內容與頭資訊的 SHA-1 校驗和，創建以該校驗和前兩個字元為名稱的子目錄，並以 (校驗和) 剩下 38 個字元為檔命名 (保存至子目錄下)。 

通過 `cat-file` 命令可以將資料內容取回。該命令是查看 Git 對象的瑞士軍刀。傳入 `-p` 參數可以讓 `cat-file` 命令輸出資料內容的類型： 

	$ git cat-file -p d670460b4b4aece5915caf5c68d12f560a9fe3e4
	test content

可以往 Git 中添加更多內容並取回了。也可以直接添加檔案。比方說可以對一個檔進行簡單的版本控制。首先，創建一個新檔，並把檔案內容儲存到資料庫中：

	$ echo 'version 1' > test.txt
	$ git hash-object -w test.txt
	83baae61804e65cc73a7201a7252750c76066a30

接著往該檔中寫入一些新內容並再次保存： 

	$ echo 'version 2' > test.txt
	$ git hash-object -w test.txt
	1f7a7a472abf3dd9643fd615f6da379c4acb3e3a

資料庫中已經將檔案的兩個新版本連同一開始的內容保存下來了： 

	$ find .git/objects -type f
	.git/objects/1f/7a7a472abf3dd9643fd615f6da379c4acb3e3a
	.git/objects/83/baae61804e65cc73a7201a7252750c76066a30
	.git/objects/d6/70460b4b4aece5915caf5c68d12f560a9fe3e4

再將檔案修復到第一個版本： 

	$ git cat-file -p 83baae61804e65cc73a7201a7252750c76066a30 > test.txt
	$ cat test.txt
	version 1

或恢復到第二個版本： 

	$ git cat-file -p 1f7a7a472abf3dd9643fd615f6da379c4acb3e3a > test.txt
	$ cat test.txt
	version 2

需要記住的是幾個版本的檔 SHA-1 值可能與實際的值不同，其次，儲存的並不是檔案名而僅僅是檔案內容。這種物件類型稱為 blob 。通過傳遞 SHA-1 值給 `cat-file -t` 命令可以讓 Git 返回任何物件的類型： 

	$ git cat-file -t 1f7a7a472abf3dd9643fd615f6da379c4acb3e3a
	blob

## Tree 物件

接下去來看 tree 物件，tree 物件可以儲存檔案名，同時也允許將一組檔案儲存在一起。Git 以一種類似 UNIX 檔案系統但更簡單的方式來儲存內容。所有內容以 tree 或 blob 物件儲存，其中 tree 物件對應於 UNIX 中的目錄，blob 物件則大致對應於 inodes 或檔案內容。一個單獨的 tree 物件包含一條或多條 tree 記錄，每一條記錄含有一個指向 blob 或子 tree 物件的 SHA-1 指標，並附有該物件的許可權模式 (mode)、類型和檔案名資訊。以 simplegit 專案為例，最新的 tree 可能是這個樣子： 

	$ git cat-file -p master^{tree}
	100644 blob a906cb2a4a904a152e80877d4088654daad0c859      README
	100644 blob 8f94139338f9404f26296befa88755fc2598c289      Rakefile
	040000 tree 99f1a6d12cb4b6f19c8655fca46c3ecf317074e0      lib

`master^{tree}` 表示 `master` 分支上最新提交指向的 tree 物件。請注意 `lib` 子目錄並非一個 blob 物件，而是一個指向別一個 tree 物件的指標： 

	$ git cat-file -p 99f1a6d12cb4b6f19c8655fca46c3ecf317074e0
	100644 blob 47c6340d6459e05787f644c2447d2595f5d3a54b      simplegit.rb

從概念上來講，Git 保存的資料如圖 9-1 所示。 


![](http://git-scm.com/figures/18333fig0901-tn.png)

Figure 9-1. Git 物件模型的簡化版

你可以自己創建 tree 。通常 Git 根據你的暫存區域或 index 來創建並寫入一個 tree 。因此要創建一個 tree 物件的話首先要通過將一些檔暫存從而創建一個 index 。可以使用 plumbing 命令 `update-index` 為一個單獨檔 ── test.txt 檔的第一個版本 ──　創建一個 index。通過該命令人工地將 test.txt 檔的首個版本加入到了一個新的暫存區域中。由於該檔原先並不在暫存區域中 (甚至就連暫存區域也還沒被創建出來呢)，必須傳入 `--add` 參數；由於要添加的檔並不在目前的目錄下而是在資料庫中，必須傳入 `--cacheinfo` 參數。同時指定檔案模式，SHA-1 值和檔案名：

	$ git update-index --add --cacheinfo 100644 \
	  83baae61804e65cc73a7201a7252750c76066a30 test.txt

在本例中，指定了檔案模式為 `100644`，表明這是一個普通檔。其他可用的模式有：`100755` 表示可執行檔，`120000` 表示符號連結(symbolic link)。檔案模式是從一般的 UNIX 檔案模式中參考來的，但是沒有那麼靈活 ── 上述三種模式僅對 Git 中的檔案 (blobs) 有效 (雖然也有其他模式用於目錄和子模組)。 

現在可以用 `write-tree` 命令將暫存區域的內容寫到一個 tree 物件了。無需 `-w` 參數 ── 如果目標 tree 不存在，呼叫 `write-tree` 會自動根據 index 狀態創建一個 tree 物件。 

	$ git write-tree
	d8329fc1cc938780ffdd9f94e0d364e0ea74f579
	$ git cat-file -p d8329fc1cc938780ffdd9f94e0d364e0ea74f579
	100644 blob 83baae61804e65cc73a7201a7252750c76066a30      test.txt

可以驗證這確實是一個 tree 物件： 

	$ git cat-file -t d8329fc1cc938780ffdd9f94e0d364e0ea74f579
	tree

再根據 test.txt 的第二個版本以及一個新檔創建一個新 tree 物件： 

	$ echo 'new file' > new.txt
	$ git update-index test.txt
	$ git update-index --add new.txt

這時暫存區域中包含了 test.txt 的新版本及一個新檔 new.txt 。創建 (寫) 該 tree 物件 (將暫存區域或 index 狀態寫入到一個 tree 物件)，然後瞧瞧它的樣子： 

	$ git write-tree
	0155eb4229851634a0f03eb265b69f5a2d56f341
	$ git cat-file -p 0155eb4229851634a0f03eb265b69f5a2d56f341
	100644 blob fa49b077972391ad58037050f2a75f74e3671e92      new.txt
	100644 blob 1f7a7a472abf3dd9643fd615f6da379c4acb3e3a      test.txt

請注意該 tree 物件包含了兩個檔案記錄，且 test.txt 的 SHA 值是早先值的 “第二版” (`1f7a7a`)。來點更有趣的，你將把第一個 tree 物件作為一個子目錄加進該 tree 中。可以用 `read-tree` 命令將 tree 物件讀到暫存區域中去。在這時，通過傳一個 `--prefix` 參數給 `read-tree`，將一個已有的 tree 物件作為一個子 tree 讀到暫存區域中： 

	$ git read-tree --prefix=bak d8329fc1cc938780ffdd9f94e0d364e0ea74f579
	$ git write-tree
	3c4e9cd789d88d8d89c1073707c3585e41b0e614
	$ git cat-file -p 3c4e9cd789d88d8d89c1073707c3585e41b0e614
	040000 tree d8329fc1cc938780ffdd9f94e0d364e0ea74f579      bak
	100644 blob fa49b077972391ad58037050f2a75f74e3671e92      new.txt
	100644 blob 1f7a7a472abf3dd9643fd615f6da379c4acb3e3a      test.txt

如果從剛寫入的新 tree 物件創建一個工作目錄，將得到位於工作目錄頂級的兩個檔和一個名為 `bak` 的子目錄，該子目錄包含了 test.txt 檔的第一個版本。可以將 Git 用來包含這些內容的資料想像成如圖 9-2 所示的樣子。 


![](http://git-scm.com/figures/18333fig0902-tn.png)

Figure 9-2. 當前 Git 資料的內容結構 

## Commit 物件

你現在有三個 tree 物件，它們指向了你要跟蹤的專案的不同快照，可是先前的問題依然存在：必須記往三個 SHA-1 值以獲得這些快照。你也沒有關於誰、何時以及為何保存了這些快照的資訊。commit 物件為你保存了這些基本資訊。 

要創建一個 commit 物件，使用 `commit-tree` 命令，指定一個 tree 的 SHA-1，如果有任何前繼提交物件，也可以指定。從你寫的第一個 tree 開始： 

	$ echo 'first commit' | git commit-tree d8329f
	fdf4fc3344e67ab068f836878b6c4951e3b15f3d

通過 `cat-file` 查看這個新 commit 物件： 

	$ git cat-file -p fdf4fc3
	tree d8329fc1cc938780ffdd9f94e0d364e0ea74f579
	author Scott Chacon <schacon@gmail.com> 1243040974 -0700
	committer Scott Chacon <schacon@gmail.com> 1243040974 -0700

	first commit

commit 物件格式很簡單：指明了該時間點專案快照的頂層樹物件、作者/提交者資訊 (從 Git 組態設定的 `user.name` 和 `user.email` 中獲得) 以及當前時間戳記、一個空行，以及提交注釋資訊。

接著再寫入另外兩個 commit 物件，每一個都指定其之前的那個 commit 物件： 

	$ echo 'second commit' | git commit-tree 0155eb -p fdf4fc3
	cac0cab538b970a37ea1e769cbbde608743bc96d
	$ echo 'third commit'  | git commit-tree 3c4e9c -p cac0cab
	1a410efbd13591db07496601ebc7a059dd55cfe9

每一個 commit 物件都指向了你創建的樹物件快照。出乎意料的是，現在已經有了真實的 Git 歷史了，所以如果執行 `git log` 命令並指定最後那個 commit 物件的 SHA-1 便可以查看歷史：

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

真棒。你剛剛通過使用低級操作而不是那些普通命令創建了一個 Git 歷史。這基本上就是執行 `git add` 和 `git commit` 命令時 Git 進行的工作　──保存修改了的檔案的 blob，更新索引，創建 tree 物件，最後創建 commit 物件，這些 commit 物件指向了頂層 tree 物件以及先前的 commit 物件。這三類 Git 物件 ── blob，tree 以及 commit ── 都各自以檔案的方式保存在 `.git/objects` 目錄下。以下所列是目前為止範例目錄的所有物件，每個物件後面的注釋裡標明了它們保存的內容： 

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

如果你按照以上描述進行了操作，可以得到如圖 9-3 所示的物件圖。 


![](http://git-scm.com/figures/18333fig0903-tn.png)

Figure 9-3. Git 目錄下的所有物件

## 物件儲存

之前我提到當儲存資料內容時，同時會有一個檔頭被儲存起來。我們花些時間來看看 Git 是如何儲存物件的。你將看到如何通過 Ruby 指令碼語言儲存一個 blob 物件 (這裡以字串 “what is up, doc?” 為例) 。使用 `irb` 命令進入 Ruby 互動式模式： 

	$ irb
	>> content = "what is up, doc?"
	=> "what is up, doc?"

Git 以物件類型為起始內容構造一個檔頭，本例中是一個 blob。然後添加一個空格，接著是資料內容的長度，最後是一個空位元組 (null byte)： 

	>> header = "blob #{content.length}\0"
	=> "blob 16\000"

Git 將檔頭與原始資料內容拼接起來，並計算拼接後的新內容的 SHA-1 校驗和。可以在 Ruby 中使用 `require` 語句導入 SHA1 digest 程式庫，然後呼叫 `Digest::SHA1.hexdigest()` 方法計算字串的 SHA-1 值： 

	>> store = header + content
	=> "blob 16\000what is up, doc?"
	>> require 'digest/sha1'
	=> true
	>> sha1 = Digest::SHA1.hexdigest(store)
	=> "bd9dbf5aae1a3862dd1526723246b20206e5fc37"

Git 用 zlib 對資料內容進行壓縮，在 Ruby 中可以用 zlib 程式庫來實現。首先需要導入該程式庫，然後用 `Zlib::Deflate.deflate()` 對資料進行壓縮： 

	>> require 'zlib'
	=> true
	>> zlib_content = Zlib::Deflate.deflate(store)
	=> "x\234K\312\311OR04c(\317H,Q\310,V(-\320QH\311O\266\a\000_\034\a\235"

最後將用 zlib 壓縮後的內容寫入磁片。需要指定保存物件的路徑 (SHA-1 值的頭兩個字元作為子目錄名稱，剩餘 38 個字元作為檔案名保存至該子目錄中)。在 Ruby 中，如果子目錄不存在可以用 `FileUtils.mkdir_p()` 函數創建它。接著用 `File.open` 方法打開檔案，並用 `write()` 方法將之前壓縮的內容寫入該檔： 

	>> path = '.git/objects/' + sha1[0,2] + '/' + sha1[2,38]
	=> ".git/objects/bd/9dbf5aae1a3862dd1526723246b20206e5fc37"
	>> require 'fileutils'
	=> true
	>> FileUtils.mkdir_p(File.dirname(path))
	=> ".git/objects/bd"
	>> File.open(path, 'w') { |f| f.write zlib_content }
	=> 32

這就行了 ── 你已經創建了一個正確的 blob 物件。所有的 Git 物件都以這種方式儲存，惟一的區別是類型不同 ── 除了字串 blob，檔頭起始內容還可以是 commit 或 tree 。不過雖然 blob 幾乎可以是任意內容，commit 和 tree 的資料卻是有固定格式的。 
��來我們要保證沒有修改到 ACL 允許範圍之外的檔案。如果你的專案 `.git` 目錄裡有前面使用過的 ACL 檔，那麼以下的 `pre-commit` 腳本將執行裡面的限制規定：

	#!/usr/bin/env ruby

	$user    = ENV['USER']

	# [ insert acl_access_data method from above ]

	# only allows certain users to modify certain subdirectories in a project
	def check_directory_perms
	  access = get_acl_access_data('.git/acl')

	  files_modified = `git diff-index --cached --name-only HEAD`.split("\n")
	  files_modified.each do |path|
	    next if path.size == 0
	    has_file_access = false
	    access[$user].each do |access_path|
	    if !access_path || (path.index(access_path) == 0)
	      has_file_access = true
	    end
	    if !has_file_access
	      puts "[POLICY] You do not have access to push to #{path}"
	      exit 1
	    end
	  end
	end

	check_directory_perms

這和服務端的腳本幾乎一樣，除了兩個重要區別。第一，ACL 檔的位置不同，因為這個腳本在當前工作目錄執行，而非 Git 目錄。ACL 檔的目錄必須由這個

	access = get_acl_access_data('acl')

改成這個:

	access = get_acl_access_data('.git/acl')

另一個重要區別是獲取「被修改檔案清單」的方式。在服務端的時候使用了查看提交紀錄的方式，可是目前的提交都還沒被記錄下來呢，所以這個清單只能從暫存區域獲取。原來是這樣：

	files_modified = `git log -1 --name-only --pretty=format:'' #{ref}`

現在要用這樣：

	files_modified = `git diff-index --cached --name-only HEAD`

不同的就只有這兩點——除此之外，該腳本完全相同。一個小陷阱在於它假設在本地執行的帳戶和推送到遠端服務端的相同。如果這二者不一樣，則需要手動設置一下 `$user` 變數。

最後一項任務是檢查確認推送內容中不包含非 fast-forward 類型的索引(reference)，不過這個需求比較少見。以下情況會得到一個非 fast-forward 類型的索引，要麼在某個已經推送過的提交上做衍合，要麼從本地不同分支推送到遠端相同的分支上。

既然伺服器會告訴你不能推送非 fast-forward 內容，而且上面的掛鉤也能阻止強制的推送，唯一剩下的潛在問題就是衍合已經推送過的提交內容。

下面是一個檢查這個問題的 pre-rabase 腳本的例子。它獲取一個所有即將重寫的提交內容的清單，然後檢查它們是否在遠端的索引(reference)裡已經存在。一旦發現某個提交可以從遠端索引裡衍變過來，它就放棄衍合操作：

	#!/usr/bin/env ruby

	base_branch = ARGV[0]
	if ARGV[1]
	  topic_branch = ARGV[1]
	else
	  topic_branch = "HEAD"
	end

	target_shas = `git rev-list #{base_branch}..#{topic_branch}`.split("\n")
	remote_refs = `git branch -r`.split("\n").map { |r| r.strip }

	target_shas.each do |sha|
	  remote_refs.each do |remote_ref|
	    shas_pushed = `git rev-list ^#{sha}^@ refs/remotes/#{remote_ref}`
	    if shas_pushed.split("\n").include?(sha)
	      puts "[POLICY] Commit #{sha} has already been pushed to #{remote_ref}"
	      exit 1
	    end
	  end
	end

這個腳本利用了一個第六章「修訂版本選擇」一節中不曾提到的語法。執行這個命令可以獲得一個所有已經完成推送的提交的列表：

	git rev-list ^#{sha}^@ refs/remotes/#{remote_ref}

`SHA^@` 語法解析該次提交的所有祖先。我們尋找任何一個提交，這個提交可以從遠端最後一次提交衍變獲得(reachable)，但從我們嘗試推送的任何一個提交的 SHA 值的任何一個祖先都無法衍變獲得——也就是 fast-forward 的內容。

這個解決方案的缺點在於它可能會很慢而且通常是沒有必要的——只要不用 -f 來強制推送，伺服器會自動給出警告並且拒絕推送內容。然而，這是個不錯的練習，而且理論上能幫助用戶避免一個將來不得不回頭修改的衍合操作。
