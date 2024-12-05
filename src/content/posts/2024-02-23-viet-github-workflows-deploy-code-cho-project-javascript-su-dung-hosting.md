---
published: 2023-07-02
title: Viết Github Workflows deploy code cho project Javascript sử dụng hosting
---
Đợt vừa rồi mình có làm một project cá nhân nho nhỏ, sau khi lựa chọn 7749 cái công nghệ thì mình quyết định chọn Strapi (React + Node) và Nuxt để làm một website bán hàng với các chức năng bán hàng cơ bản. Điểm chung của 2 framework này đó là đều sử dụng Javascript/Typescript nên cũng khá tiện trong quá trình phát triển.

Cũng trong thời gian này mình cũng hốt được cái deal hosting khá hời nên cũng định tìm hiểu qua cách viết Github Workflows để tự động deploy code mới nhất lên hosting khi có người push code lên branch `master`.

Trước khi viết mình đã đơn giản suy nghĩ đến luồng cơ bản của đó là sử dụng một thư viện SSH kết nối đến hosting, đi đến thư mục của của project và `git pull && yarn` thôi. Từ đó mình cho ra cái workflow như thế này:

    name: Deploy
    
    on:
      push:
        branches: [master]
    
    jobs:
      Deploy:
        runs-on: ubuntu-latest
        steps:
          - name: Main workflow
            uses: appleboy/ssh-action@master
            with:
              host: ${{ secrets.SSH_HOST }}
              username: ${{ secrets.SSH_USERNAME }}
              password: ${{ secrets.SSH_PASSWORD }}
              port: ${{ secrets.SSH_PORT }}
              script: |
                cd ${{ secrets.PATH_TO_PROJECT }}
                git pull
                yarn install
                yarn build
                # ...
    

Nhưng sau khi test thử thì mình nhận ra vấn đề:

* Workflow không thể trigger thủ công trên Github.

* Workflow có thể đồng thời chạy nhiều instance cùng lúc, điều này gây ra vấn đề không đồng nhất vì workflow thao tác trực tiếp vào hosting.

Do đó mình chỉnh sửa lại như sau:

    name: Deploy
    
    on:
      push:
        branches: [master]
      workflow_dispatch: # Để có thể trigger workflow trên Github
    
    concurrency:
      # Đặt tên group cho các instance, các instance có cùng tên group
      # sẽ không được chạy cùng thời điểm. Ở đây mình lấy tên group = tên của     
      # workflow + tên branch nên mỗi branch chỉ có một instance chạy
      # tại cùng một thời điểm
      group: ${{ github.workflow }}-${{ github.ref }}
      # Nếu trong cùng một group mà có nhiều instance được tạo ra thì sẽ dừng
      # các instance cũ hơn lại và chạy instance mới nhất
      cancel-in-progress: true
    
    # ...
    

Nhìn có vẻ ổn rồi nhưng lại xảy ra vấn đề. Khi workflow chạy đến câu lệnh `yarn install && yarn build` thì các website đặt trên hosting của mình đều không truy cập được do câu lệnh trên đã sử dụng hết số lượng process có thể tạo ra rồi, ngoài ra còn đẩy RAM lên gần chạm limit nữa.

Thế nên mình đã nảy ra ý tưởng tại sao ta không chuyển phần build code sang môi trường của workflow và sau đó chỉ cần upload đống code đã build lên hosting rồi run thôi? Nghĩ như thế nên mình chỉnh sửa lại workflow như sau:

    name: CD
    
    # ...
    
    jobs:
      deploy:
        name: Deploy
        runs-on: ubuntu-latest
        env:
          ZIP_FILE: ${{ github.event.repository.name }}-${{ github.ref_name }}.zip
        steps:
          - name: Checkout
            uses: actions/checkout@v3
    
          - name: Run install
            run: yarn install --frozen-lockfile --silent
    
          - name: Build code
            run: yarn build
            env:
              NODE_ENV: production
    
          - name: Compress
            run: rm -rf node_modules && zip -qr $ZIP_FILE .
    
          - name: Configure SSH
            run: |
              mkdir -p ~/.ssh/
              echo "$SSH_KEY" > ~/.ssh/server
              chmod 600 ~/.ssh/server
              cat > ~/.ssh/config <<END
              Host server
                HostName $SSH_HOST
                User $SSH_USERNAME
                IdentityFile ~/.ssh/server
                PubkeyAuthentication yes
                ChallengeResponseAuthentication no
                PasswordAuthentication no
                StrictHostKeyChecking no
              END
            env:
              SSH_USERNAME: ${{ secrets.SSH_USERNAME }}
              SSH_KEY: ${{ secrets.SSH_KEY }}
              SSH_HOST: ${{ secrets.SSH_HOST }}
    
          - name: Upload
            run: scp $ZIP_FILE ${{ secrets.SSH_USERNAME }}@server:~
    
          - name: Migrate
            run: # ...
    

Vậy là hosting đã không cần phải gánh phần build code nữa, cũng không cần phải cài đặt Git trên hosting để có thể pull code mới nhất về nữa vì giờ đây code mới đã được upload lên hosting thông qua SSH rồi.

Nhưng sau một vài lần test nữa mình thấy rằng trong mỗi workflow instance thì khi chạy câu lệnh `yarn install` đều phải tải các thư viện về từ đầu. Điều này làm tăng thời gian chờ đợi. Do đó mình sử dụng thêm một số thư viện caching để có thể sử dụng lại những thư viện đã được tải về trong những instance trước đó.

Ngoài ra mình còn quên để ý đến một thứ nữa đó là version của node.js, để sử dụng chính xác version của node.js trên hosting để tránh việc xảy ra lỗi không cần thiết thì mình đã tạo thêm file `.nvmrc` trong project và define version mình muốn sử dụng. File `.nvmrc` này được sử dụng bởi thư viện nvm dùng để quản lý version của node.js trong hệ thống.

Kết quả sau khi thêm caching và node.js version manager ta được như sau:

    name: CD
    
    # ...
    
    jobs:
      deploy:
        name: Deploy
        runs-on: ubuntu-latest
        env:
          ZIP_FILE: ${{ github.event.repository.name }}-${{ github.ref_name }}.zip
        steps:
          - name: Checkout
            uses: actions/checkout@v3
    
          # nvm
          - uses: actions/setup-node@v3
            with:
              node-version-file: ".nvmrc"
          
          # Lấy đường dẫn của yarn cache trong workflow instance
          - name: Get yarn cache directory path
            id: yarn-cache-dir-path
            run: echo "dir=$(yarn cache dir)" >> $GITHUB_OUTPUT
    
          # Caching
          - uses: actions/cache@v3
            id: yarn-cache
            with:
              # Đường dẫn đến yarn cache path đã lấy bên trên
              path: ${{ steps.yarn-cache-dir-path.outputs.dir }}
              # Sử dụng key để quản lý version của cache, ở đây thư viện sẽ tạo
              # cache mới khi mà file `yarn.lock` thay đổi. Việc sử dụng key này
              # tương tự như Redis
              key: ${{ runner.os }}-yarn-${{ hashFiles('**/yarn.lock') }}
              # Tiền tố của các key thay thế nếu key bên trên không tồn tại
              restore-keys: |
                ${{ runner.os }}-yarn-
    
          # Thư viện này hỗ trợ việc chạy các câu lệnh của yarn. Ngoài ra còn
          # hỗ trợ caching và node.js version manager tương tự như 2 thư viện
          # bên trên. Có thể sử dụng thay thế.
          - name: Run install
            uses: borales/actions-yarn@v4
            with:
              cmd: install --frozen-lockfile --silent
    
          - name: Build code
            run: yarn build
            env:
              NODE_ENV: production
    
          - name: Compress
            run: rm -rf node_modules && zip -qr $ZIP_FILE .
    
          - name: Configure SSH
            run: |
              mkdir -p ~/.ssh/
              echo "$SSH_KEY" > ~/.ssh/server
              chmod 600 ~/.ssh/server
              cat > ~/.ssh/config <<END
              Host server
                HostName $SSH_HOST
                User $SSH_USERNAME
                IdentityFile ~/.ssh/server
                PubkeyAuthentication yes
                ChallengeResponseAuthentication no
                PasswordAuthentication no
                StrictHostKeyChecking no
              END
            env:
              SSH_USERNAME: ${{ secrets.SSH_USERNAME }}
              SSH_KEY: ${{ secrets.SSH_KEY }}
              SSH_HOST: ${{ secrets.SSH_HOST }}
    
          - name: Upload
            run: scp $ZIP_FILE ${{ secrets.SSH_USERNAME }}@server:~
    
          - name: Migrate
            run: # ...
    

Và kết quả sau khi đã thêm caching thì mình đã giảm thời gian chạy workflow trung bình từ 2 phút 31 giây xuống còn … 2 phút 16 giây. Cũng gọi là có chút nhanh hơn rồi đấy! 😀

Hiện tại mình chỉ mới tìm hiểu đến đấy, sau này do trải nghiệm không tốt lắm với Strapi nên mình đã chuyển sang Laravel và viết lại một workflow mới. Nhưng cũng từ việc này mình cũng đã có thêm một chút kiến thức về Github Workflow và hosting. Kiến thức của mình có hạn nên nếu có thiếu sót mong các bạn góp ý thêm nhé!

Cám ơn các bạn đã theo dõi!
