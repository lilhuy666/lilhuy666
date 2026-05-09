def db_get_history_for_chart(email, car_name, days):
    conn = get_db_connection()
    try:
        cursor = conn.cursor()
        query = """
            SELECT DATE(date) as day, AVG(consumption) as avg_consumption
            FROM history 
            WHERE email = %s 
              AND date >= CURRENT_DATE - INTERVAL %s
        """
        params = [email, f'{days} days']
        
        if car_name and car_name != "— Без авто —" and car_name != "— No car —":
            query += " AND car_name = %s"
            params.append(car_name)
        
        query += " GROUP BY DATE(date) ORDER BY day"
        
        cursor.execute(query, params)
        return [(r[0], r[1]) for r in cursor.fetchall()]
    finally:
        put_db_connection(conn)
