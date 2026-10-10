/*
 * ============================================================================
 * ESP-IDF 组件 CI 模板使用指南
 * ============================================================================
 *
 * 本仓库提供了一套可复用的 GitHub Actions 工作流，用于 ESP-IDF 组件的
 * 构建、主机端测试和上传至组件注册表。
 * 你可以在自己的组件仓库中通过 uses 直接调用这些工作流，无需重复编写 CI 逻辑。
 *
 * ----------------------------------------------------------------------------
 * 包含的工作流
 * ----------------------------------------------------------------------------
 *
 * 文件          | 显示名称               | 功能
 * --------------|------------------------|------------------------------------
 * build.yml     | ESP-IDF 固件构建       | 使用官方 ESP-IDF Docker 镜像编译指定
 *               |                        | 示例，支持多目标（esp32c3、esp32s3
 *               |                        | 等），并缓存 ccache 加速
 * test.yml      | 主机端单元测试         | 在 Ubuntu 上安装 CMake、G++、
 *               |                        | Ninja，进入 examples 目录配置、
 *               |                        | 构建并运行 CTest
 * upload.yml    | 上传组件到组件注册表   | 将当前仓库作为组件上传到
 *               |                        | ESP Component Registry
 *
 * 所有工作流均通过 workflow_call 触发，只能被其他工作流调用，不能直接手动运行。
 *
 * ----------------------------------------------------------------------------
 * 一、组件仓库结构建议
 * ----------------------------------------------------------------------------
 *
 * 为了让这些工作流正常工作，你的组件仓库建议采用以下目录结构：
 *
 * your-component/
 * ├── .github/
 * │   └── workflows/
 * │       └── ci.yml                 # 你自己的主工作流，调用本仓库的模板
 * ├── examples/
 * │   ├── CMakeLists.txt             # 主机端测试的 CMake 入口（供 test.yml 使用）
 * │   └── basic/                     # ESP-IDF 示例项目（供 build.yml 使用）
 * │       ├── CMakeLists.txt
 * │       └── main/
 * │           └── main.c
 * ├── include/
 * │   └── your_component.h
 * ├── src/
 * │   └── your_component.c
 * ├── test/                          # 可选：测试源码，由 examples/CMakeLists.txt 引入
 * │   └── test_main.cpp
 * ├── CMakeLists.txt                 # 组件根 CMakeLists.txt
 * ├── idf_component.yml              # 组件清单（upload.yml 需要）
 * └── README.md
 *
 * 关键说明
 * --------
 * - examples/basic：一个完整的 ESP-IDF 示例工程，包含自己的 CMakeLists.txt
 *   和 main/。build.yml 默认会进入这个目录执行 idf.py build。
 * - examples/CMakeLists.txt：test.yml 会进入 examples 目录并执行
 *   cmake -B build，因此这里必须有一个可独立配置的 CMake 项目，用于构建和
 *   运行主机端测试。你可以在这个 CMakeLists.txt 中通过 add_subdirectory
 *   引入 ../test 目录的测试代码。
 * - idf_component.yml：组件清单文件，upload.yml 依赖它来上传组件。如果缺少，
 *   上传会失败。
 *
 * 如果你的测试目录不在 examples 下，可以 fork 本仓库并修改 test.yml，
 * 或者在自己的工作流中直接编写测试命令。
 *
 * ----------------------------------------------------------------------------
 * 二、如何调用这些工作流
 * ----------------------------------------------------------------------------
 *
 * 在你的组件仓库中创建 .github/workflows/ci.yml，按需组合调用。
 *
 * 1. 调用 build.yml（ESP-IDF 固件构建）
 * ---------------------------------------
 *
 * jobs:
 *   build:
 *     uses: smalllin0/my-ci-templates/.github/workflows/build.yml@master
 *     with:
 *       esp_idf_version: 'v6.1'          # 与 Docker 镜像 tag 一致
 *       targets: '["esp32c3","esp32s3"]' # JSON 数组字符串
 *       example_path: 'examples/basic'   # 示例工程路径
 *
 * 输入参数：
 *
 * 参数              | 类型   | 默认值            | 说明
 * ------------------|--------|-------------------|--------------------------------
 * esp_idf_version   | string | v6.1              | ESP-IDF Docker 镜像版本，如
 *                   |        |                   | v6.1、v5.4
 * targets           | string | ["esp32c3"]       | JSON 数组字符串，指定要构建的
 *                   |        |                   | 芯片目标
 * example_path      | string | examples/basic    | 示例工程所在目录（相对于仓库根）
 *
 * 功能：
 * - 自动拉取代码及子模块
 * - 缓存 ~/.ccache，加速二次编译
 * - 使用 espressif/idf:${inputs.esp_idf_version} 容器
 * - 对每个 target 执行 idf.py set-target 和 idf.py build
 *
 * 2. 调用 test.yml（主机端单元测试）
 * -----------------------------------
 *
 * jobs:
 *   test:
 *     uses: smalllin0/my-ci-templates/.github/workflows/test.yml@master
 *
 * 输入参数：无（当前版本不接受输入）
 *
 * 功能：
 * - 在 Ubuntu 上安装 cmake、g++、ninja-build
 * - 进入 examples 目录，删除旧的 build
 * - 执行 cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=Debug
 * - 执行 cmake --build examples/build
 * - 执行 ctest --test-dir examples/build --output-on-failure
 *
 * 前提：仓库的 examples 目录下必须有 CMakeLists.txt，且能正确配置和运行测试。
 *
 * 3. 调用 upload.yml（上传组件到注册表）
 * ---------------------------------------
 *
 * jobs:
 *   upload:
 *     uses: smalllin0/my-ci-templates/.github/workflows/upload.yml@master
 *     with:
 *       namespace: your-namespace          # 必填，组件命名空间
 *     secrets:
 *       IDF_COMPONENT_API_TOKEN: ${{ secrets.IDF_COMPONENT_API_TOKEN }}
 *
 * 输入参数：
 *
 * 参数       | 类型   | 必填 | 说明
 * -----------|--------|------|------------------------------------------
 * namespace  | string | 是   | 组件在注册表中的命名空间（用户名或组织名）
 *
 * Secrets：
 *
 * 名称                      | 必填 | 说明
 * --------------------------|------|----------------------------------------
 * IDF_COMPONENT_API_TOKEN   | 是   | 从 ESP Component Registry 获取的 API Token
 *
 * 功能：
 * - 使用 espressif/idf:v6.1 容器
 * - 加载 ESP-IDF 环境
 * - 执行 compote component upload --namespace <namespace> --name <仓库名>
 *
 * 请确保仓库根目录有 idf_component.yml，且组件版本号已正确设置。
 *
 * ----------------------------------------------------------------------------
 * 三、完整调用示例
 * ----------------------------------------------------------------------------
 *
 * 以下是一个完整的 .github/workflows/ci.yml，展示了如何组合使用三个工作流：
 *
 * name: CI
 *
 * on:
 *   push:
 *     branches: [master, main]
 *   pull_request:
 *
 * jobs:
 *   build:
 *     uses: smalllin0/my-ci-templates/.github/workflows/build.yml@master
 *     with:
 *       esp_idf_version: 'v6.1'
 *       targets: '["esp32c3","esp32s3"]'
 *       example_path: 'examples/basic'
 *
 *   test:
 *     uses: smalllin0/my-ci-templates/.github/workflows/test.yml@master
 *
 *   upload:
 *     needs: [build, test]
 *     if: github.event_name == 'push' && startsWith(github.ref, 'refs/tags/v')
 *     uses: smalllin0/my-ci-templates/.github/workflows/upload.yml@master
 *     with:
 *       namespace: your-namespace
 *     secrets:
 *       IDF_COMPONENT_API_TOKEN: ${{ secrets.IDF_COMPONENT_API_TOKEN }}
 *
 * 说明：
 * - build 和 test 在每次 push 和 PR 时运行。
 * - upload 仅在打 tag（如 v1.0.0）时运行，并且需要 build 和 test 成功。
 * - 记得在仓库的 Settings → Secrets and variables → Actions 中添加
 *   IDF_COMPONENT_API_TOKEN。
 *
 * ----------------------------------------------------------------------------
 * 四、注意事项
 * ----------------------------------------------------------------------------
 *
 * 1. 版本固定
 *    示例中使用 @master，建议在生产环境中替换为具体的 tag 或 commit SHA，
 *    例如 @v1.0.0，避免上游变更导致 CI 意外失败。
 *
 * 2. esp_idf_version 与 Docker 镜像
 *    该参数直接用作 espressif/idf: 的 tag，请确保填写的版本在 Docker Hub
 *    上存在，例如 v6.1、v5.4。不要写成 v6.1.0。
 *
 * 3. targets 必须是 JSON 数组字符串
 *    例如 '["esp32c3","esp32s3"]'，外层单引号是 YAML 语法，内层是 JSON。
 *
 * 4. test.yml 的目录假设
 *    当前 test.yml 硬编码了 examples 目录。如果你的测试项目不在 examples 下，
 *    需要修改工作流或自行编写测试步骤。
 *
 * 5. upload.yml 的组件名称
 *    上传时使用的组件名称取自 github.event.repository.name，即仓库名。
 *    请确保仓库名与 idf_component.yml 中的组件名一致。
 *
 * 6. ccache 缓存
 *    build.yml 使用 actions/cache 缓存 ~/.ccache，key 包含 target 和
 *    commit SHA。首次构建后，后续构建会显著加快。
 *
 * 7. 子模块
 *    build.yml 在 checkout 时启用了 submodules: recursive，如果你的项目
 *    有子模块，请确保 .gitmodules 配置正确。
 *
 * ----------------------------------------------------------------------------
 * 五、常见问题
 * ----------------------------------------------------------------------------
 *
 * Q：为什么 test.yml 要进入 examples 目录？
 * A：当前设计假设 examples 目录是一个 CMake 项目，用于构建主机端测试。
 *    你可以将测试代码放在 examples 下，或通过 examples/CMakeLists.txt
 *    引入 ../test 中的测试源码。
 *
 * Q：上传失败提示 token 无效？
 * A：请检查 IDF_COMPONENT_API_TOKEN 是否已在调用方仓库的 Secrets 中正确
 *    设置，并且该 Token 具有上传权限。
 *
 * Q：如何只构建特定 target？
 * A：在调用 build.yml 时传入 targets 参数，例如 targets: '["esp32c3"]'。
 *
 * Q：可以同时运行多个 target 吗？
 * A：可以，build.yml 使用矩阵策略，每个 target 会并行构建。传入 JSON 数组
 *    即可，如 '["esp32c3","esp32s3"]'。
 *
 * ----------------------------------------------------------------------------
 * 如有问题或需要定制，请提交 Issue 或 PR。
 * ============================================================================
 */