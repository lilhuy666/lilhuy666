@app.route('/calculate', methods=['POST'])
def calculate():
    """Calculate fuel consumption and cost"""
    try:
        dist = float(request.form.get('distance', 0))
        price = float(request.form.get('price', 0))
    except:
        return jsonify({'error': tr('enter_numbers')}), 400

    if dist <= 0:
        return jsonify({'error': tr('distance_positive')}), 400

    if price <= 0:
        return jsonify({'error': tr('price_positive')}), 400

    mode = request.form.get('mode', 'consumption')
    car_name = request.form.get('car', tr('no_car')) if 'user' in session else tr('no_car')
    currency = session.get('currency', '₽ RUB') if 'user' in session else '₽ RUB'
    currency_symbol = CURRENCIES.get(currency, '₽')

    if mode == 'consumption':
        try:
            fuel = float(request.form.get('fuel', 0))
        except:
            return jsonify({'error': tr('enter_numbers')}), 400

        if fuel <= 0:
            return jsonify({'error': tr('fuel_positive')}), 400

        consumption = (fuel / dist) * 100
        cost = fuel * price
    else:
        try:
            avg_input = float(request.form.get('avg_consumption', 0))
        except:
            return jsonify({'error': tr('enter_numbers')}), 400

        if avg_input <= 0:
            return jsonify({'error': tr('distance_positive2')}), 400

        consumption = avg_input
        fuel = (avg_input / 100) * dist
        cost = ((avg_input / 100) * dist) * price

    if 'user' in session:
        try:
            db_add_history(session['user'], {
                'date': datetime.now().strftime("%d.%m.%Y %H:%M"),
                'car': car_name,
                'distance': dist,
                'fuel': round(fuel, 2),
                'price': price,
                'currency': currency_symbol,
                'consumption': round(consumption, 2),
                'cost': round(cost, 2)
            })
            if car_name not in (tr('no_car'), '-- No car --', '— Без авто —', '— No car —'):
                db_update_avg(session['user'], car_name)
        except:
            pass

    return jsonify({
        'consumption': round(consumption, 2),
        'fuel': round(fuel, 2),
        'cost': round(cost, 2),
        'mode': mode,
        'currency': currency_symbol
    })
