Para consultar dados de quais vendors tem qual metodo de pagamento
usar a tabela vendor_payment_methods do vsp_payments.


 -- Vendors que efetivamente tiveram pagamentos ACH processados
  SELECT
      p.vendor_code,
      MIN(pay.vendor_name) AS vendor_name,
      COUNT(*) AS total_ach_payments,
      SUM(CASE WHEN p.status IN ('CAPTURED','PAID') THEN 1 ELSE 0 END) AS successful,
      SUM(CASE WHEN p.status = 'CANCELLED' THEN 1 ELSE 0 END) AS cancelled,
      SUM(CASE WHEN p.status = 'PROCESSING' THEN 1 ELSE 0 END) AS processing,
      FORMAT(SUM(CASE WHEN p.status IN ('CAPTURED','PAID') THEN p.total_amount ELSE 0 END), 'N2') AS total_paid,
      MIN(p.created_at) AS first_payment,
      MAX(p.created_at) AS last_payment
  FROM payments p
  LEFT JOIN payables pay ON pay.vendor_code = p.vendor_code AND pay.deleted_at IS NULL
  WHERE p.deleted_at IS NULL AND p.payment_method = 'ACH'
  GROUP BY p.vendor_code
  ORDER BY COUNT(*) DESC