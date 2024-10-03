# test_task

import csv
from django.db import models, transaction
from django.utils import timezone
from django.http import HttpResponse
from rest_framework import serializers, status
from rest_framework.views import APIView
from rest_framework.response import Response


# модели
class Player(models.Model):
    """
    Модель игрока. Содержит информацию о первом входе, ежедневных очках и бонусах.
    """
    username = models.CharField(max_length=100, unique=True)
    first_login = models.DateTimeField(default=timezone.now)
    points = models.PositiveIntegerField(default=0)

    def __str__(self):
        return self.username


class Boost(models.Model):
    """
    Модель бонуса (буста). Хранит информацию о типе буста и его описании.
    """
    name = models.CharField(max_length=100)
    boost_type = models.CharField(max_length=100)
    description = models.TextField()

    def __str__(self):
        return self.name


class PlayerBoost(models.Model):
    """
    Связь игрока с бустами, фиксирующая момент назначения бонуса
    """
    player = models.ForeignKey(Player, on_delete=models.CASCADE)
    boost = models.ForeignKey(Boost, on_delete=models.CASCADE)
    assigned_at = models.DateTimeField(default=timezone.now)


class Level(models.Model):
    """
    Модель уровня
    """
    title = models.CharField(max_length=100)
    order = models.IntegerField(default=0)

    def __str__(self):
        return self.title


class Prize(models.Model):
    """
    Призы, которые присваиваются за прохождение уровней
    """
    title = models.CharField(max_length=100)

    def __str__(self):
        return self.title


class PlayerLevel(models.Model):
    """
    Модель связывает игрока с уровнем, хранит информацию о том, был ли пройден уровень, и результаты
    """
    player = models.ForeignKey(Player, on_delete=models.CASCADE)
    level = models.ForeignKey(Level, on_delete=models.CASCADE)
    completed = models.DateField(null=True, blank=True)
    is_completed = models.BooleanField(default=False)
    score = models.PositiveIntegerField(default=0)

    def __str__(self):
        return f"{self.player.username} - {self.level.title}"


class LevelPrize(models.Model):
    """
    Модель связывает уровень с призом, фиксируя факт получения приза за прохождение уровня
    """
    level = models.ForeignKey(Level, on_delete=models.CASCADE)
    prize = models.ForeignKey(Prize, on_delete=models.CASCADE)
    received = models.DateField(null=True, blank=True)


# логика для работы с данными
class PlayerService:

    @staticmethod
    def assign_prize_to_player(player_id, level_id, prize_id):
        """
        Присваивает приз игроку за завершение уровня, если уровень пройден
        """
        try:
            # Получаем PlayerLevel для игрока и уровня
            player_level = PlayerLevel.objects.select_related('player', 'level').get(
                player__id=player_id,
                level_id=level_id
            )
            prize = Prize.objects.get(id=prize_id)

            # Проверяем, завершён ли уровень
            if player_level.is_completed:
                # Проверяем, был ли уже присвоен приз
                if not LevelPrize.objects.filter(level=player_level.level, prize=prize).exists():
                    # Используем транзакцию для атомарности
                    with transaction.atomic():
                        LevelPrize.objects.create(
                            level=player_level.level,
                            prize=prize,
                            received=timezone.now()
                        )
                    return {"status": "Prize assigned"}
                return {"status": "Prize already assigned"}
            else:
                return {"status": "Level not completed"}
        except PlayerLevel.DoesNotExist:
            return {"error": "Player or Level not found"}
        except Prize.DoesNotExist:
            return {"error": "Prize not found"}
        except Exception as e:
            return {"error": str(e)}

    @staticmethod
    def export_player_data():
        """
        Экспорт данных о пройденных уровнях и присвоенных призах
        """
        data = []
        player_levels = PlayerLevel.objects.select_related('player', 'level').prefetch_related('level__levelprize_set')

        for pl in player_levels:
            prize = pl.level.levelprize_set.first().prize.title if pl.level.levelprize_set.exists() else 'None'
            data.append({
                'player_id': pl.player.id,
                'level_title': pl.level.title,
                'is_completed': pl.is_completed,
                'prize': prize
            })
        return data


# сериалайзеры
class PrizeAssignmentSerializer(serializers.Serializer):
    """
    Сериалайзер для присвоения приза игроку за завершение уровня
    """
    player_id = serializers.IntegerField()
    level_id = serializers.IntegerField()
    prize_id = serializers.IntegerField()

    def create(self, validated_data):
        """
        Присваивает приз игроку с помощью сервиса.
        """
        return PlayerService.assign_prize_to_player(
            player_id=validated_data['player_id'],
            level_id=validated_data['level_id'],
            prize_id=validated_data['prize_id']
        )


# вьшки
class AssignPrizeView(APIView):
    """
    API для присвоения приза игроку за прохождение уровня
    """
    def post(self, request, *args, **kwargs):
        serializer = PrizeAssignmentSerializer(data=request.data)
        if serializer.is_valid():
            result = serializer.save()
            if 'error' in result:
                return Response(result, status=status.HTTP_400_BAD_REQUEST)
            return Response(result, status=status.HTTP_200_OK)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)


class ExportCSVView(APIView):
    """
    API для экспорта данных о прохождении уровней и призах в формате CSV
    """
    def get(self, request, *args, **kwargs):
        # Создаём ответ в виде CSV
        response = HttpResponse(content_type='text/csv')
        response['Content-Disposition'] = 'attachment; filename="player_level_data.csv"'

        writer = csv.writer(response)
        writer.writerow(['Player ID', 'Level Title', 'Completed', 'Prize'])

        # Получаем данные через сервис
        data = PlayerService.export_player_data()

        for row in data:
            writer.writerow([row['player_id'], row['level_title'], row['is_completed'], row['prize']])

        return response

