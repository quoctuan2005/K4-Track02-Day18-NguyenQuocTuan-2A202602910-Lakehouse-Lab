# Reflection: Lakehouse Anti-Patterns

Trong hệ thống AI Observability và RAG phục vụ mô hình ngôn ngữ lớn (LLM) mà tôi quan tâm, anti-pattern dễ vướng nhất là **External Vector Index Drift** (lệch pha vòng đời giữa bảng Lakehouse và Vector DB ngoại vi).

Khi người dùng thực thi quyền xóa dữ liệu cá nhân (GDPR Right to Erasure) hoặc khi cập nhật tài liệu, lệnh `DELETE` chỉ cập nhật trong bảng Delta/Iceberg. Nếu thiếu cơ chế đồng bộ tự động, Vector DB ngoại vi (như Pinecone/Milvus) vẫn giữ nguyên vector embedding cũ, tiếp tục trả về dữ liệu nhạy cảm đã bị thu hồi trong các câu truy vấn Semantic Search. Điều này gây rủi ro tuân thủ nghiêm trọng và phá vỡ tính nhất quán ACID.

**Cách phòng tránh:** Kích hoạt Change Data Feed (CDF) trên Lakehouse để phát sự kiện tombstone sang Vector DB ngay khi commit hoàn tất; hoặc tận dụng In-Lakehouse Vector Search (lưu trữ vector int8 trực tiếp trong Parquet và truy vấn qua DuckDB) để đảm bảo mọi thao tác đọc đều tuân thủ ACID transaction log.

*(Phạm vi dùng AI: Tham khảo rà soát câu chữ và đối chiếu thuật ngữ kỹ thuật theo chuẩn Day 18).*
