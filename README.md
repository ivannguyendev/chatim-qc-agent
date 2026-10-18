# chatim-qc-agent

Repo QC automation viết bằng **Playwright + TypeScript**. Repo biến tài liệu yêu cầu (PRD, tính năng, API, ảnh chụp màn hình, video) thành checklist, rồi test case, rồi code test chạy tự động.

> Trạng thái: chưa khởi tạo project.

## Mục tiêu

- Dev nhận kết quả test trong vài phút sau khi tạo PR.
- Không để lọt lỗi regression nghiêm trọng lên production.
- Các luồng cốt lõi: Auth, Nhắn tin real-time (WebSocket), Nhóm/Kênh, Đính kèm file luôn hoạt động.
- Tự động hoá khoảng 80% regression để Manual QA tập trung vào exploratory test.
- Có quality gate đáng tin cậy để tự động deploy.

## Quy trình tuần tự bắt buộc

Tài liệu yêu cầu → `docs/.../checklist.md` → `docs/.../testcases.md` → Playwright spec (`tests/`).

> **Nguyên tắc:** Code test luôn phải có checklist và testcase làm căn cứ; testcase phải được sinh từ checklist. Tuyệt đối không để Agent tự lên plan rồi viết code test trực tiếp mà bỏ qua 2 tài liệu này.

Quy ước chi tiết dành cho agent nằm trong [CLAUDE.md](CLAUDE.md), cẩm nang quy chuẩn kiểm thử nằm trong [TESTING.md](TESTING.md).

## Cài đặt & chạy

```bash
npm ci
npx playwright install --with-deps
cp .env.example .env          # điền BASE_URL và tài khoản test

npm run test:smoke            # bộ smoke (chạy khi có PR)
npm run test:regression       # bộ regression (chạy hằng đêm)
npm run test:api              # test API
npm test                      # chạy toàn bộ
npx playwright show-report    # xem báo cáo
```

Chọn môi trường bằng biến `TEST_ENV` (`staging` hoặc `production`).

## Thư mục chính

- `tests/smoke/`: các luồng quan trọng
- `tests/regression/`: test chi tiết theo module
- `tests/api/`: test tầng API
- `docs/`: tài liệu yêu cầu và kịch bản hoàn chỉnh của sản phẩm (checklist, test case)
- `TESTING.md`: cẩm nang quy chuẩn kiểm thử (Checklist & Test Case standards, Playwright mappings)
- `INDEXES.csv`: chỉ mục tìm kiếm context codebase cho AI Agent (path, kind, purpose, key_symbols, used_by, tests, decisions)
